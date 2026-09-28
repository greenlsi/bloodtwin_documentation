# Plan de refactorización: API de optimizaciones, consolidación de meses y persistencia DEVS

**Estado:** borrador para discusión. Al aprobarse, cada bloque de la Sección 4 se convierte en una issue independiente en el repositorio correspondiente, referenciando este documento.

**Alcance:** `bloodtwin_backend` (orquestador/API), `bloodtwin_scheduler_milp` (OPL), `bloodtwin_devs` (simulador DEVS), `bloodtwin_database` (esquema Postgres/GCP), `bloodtwin_database_updater` (volcado mensual de histórico real), `bloodtwin_frontend_v2` (UI de optimización).

---

## 0. Evidencia en el código actual

Antes de proponer nada, esto es lo que hoy hace el código (no es interpretación, son citas de archivos reales):

- **`bloodtwin_backend/src/domain/repositories/sessions.py` → `save_simulation_run`**: cuando hay `constraints` (una iteración/optimización con restricciones), las escribe en disco:
  ```python
  base_path = os.getenv("CONSTRAINTS_BASE_PATH", "/home/greenlsi/bloodtwin_data/constraints")
  constraints_json_path = os.path.join(base_path, f"{run_id}.json")
  with open(constraints_json_path, "w", ...) as f:
      json.dump(constraints, f, ...)
  ```
  Solo la **ruta** (`constraints_json_path`) se guarda en Postgres (`scheduler_input`). El contenido vive únicamente en el filesystem del contenedor. Si el contenedor se recrea sin volumen persistente, la fila de Postgres sigue existiendo pero apunta a un fichero que ya no está.
- **Cuando no hay `constraints`** (ejecución "base" del mes), el mismo método **borra la fila anterior** de `dispatcher_processes` / `scheduler_input` / `scheduler_output` para ese mes/año y crea una nueva. No hay historial de versiones: cada recálculo del mes borra el run "base" previo.
- **Cuando sí hay `constraints`** (cada iteración/optimización), se **inserta una fila nueva** sin relación explícita "esta iteración sustituye/deriva de aquella otra", sin campo de estado y sin ningún candado que impida seguir iterando sobre un mes ya usado como base de meses posteriores.
- **No existe ningún campo de estado** (`DRAFT`/`CONSOLIDATED`/etc.) en `dispatcher_processes` ni en `scheduler_input`: solo `process_status VARCHAR(255)`, sin valores controlados ni semántica de negocio.
- **`bloodtwin_scheduler_milp/api.py`** (`POST /optimize`) lee el histórico ejecutando `csv_to_check()` sobre `data/src/historical_*.csv` (un único fichero fijo montado en el contenedor) y escribe/lee resultados en `./data/opl_data/output/optimism_free/{version}{year}{month}/optimization_result.csv`. No hay lectura desde base de datos en ningún punto de este flujo.
- **`bloodtwin_devs/src/devs_server.py`** expone `/simulate` como único endpoint operativo (aparte de `/health`); no tiene endpoints de listado, recuperación por id, ni gestión de estado. Es un motor de cálculo puro, sin persistencia propia (correcto como diseño, pero hoy nadie más que el backend orquesta su ciclo de vida).
- **El "Database Updater"** (`bloodtwin_database_updater`) ya existe y vuelca a Postgres los ficheros que llegan a un bucket GCS mediante Cloud Function — es decir, el mecanismo de ingesta de histórico real a base de datos **ya está construido** para otro flujo (fichas de donante/laboratorio) y debería reutilizarse como patrón para el histórico de scheduler/DEVS, en lugar de crear un componente nuevo desde cero.

Esta base de código confirma exactamente los tres problemas del enunciado: (1) sin control de versiones/estado de optimización, (2) sin concepto de mes consolidado, (3) incongruencia CSV-en-contenedor vs. fila-en-Postgres.

---

## Bloque A — API y flujo de consolidación

### A.1 Apificación

Hoy el "backend" (`bloodtwin_backend`) ya es una API HTTP independiente del frontend (Flask/FastAPI + Pydantic), y `bloodtwin_scheduler_milp` y `bloodtwin_devs` son también servicios HTTP propios. El problema no es que falte una API, sino que **el backend no expone un recurso "optimización" con identidad y ciclo de vida propios**; solo expone verbos de acción (`/simulate`, `/optimize`) que escriben en tablas planas sin versión.

Propuesta de recurso explícito `MonthlyOptimization`:

```
GET    /api/optimizations?center_id=&year=&month=
GET    /api/optimizations/{optimization_id}
POST   /api/optimizations                      # crea un DRAFT para un mes (dispara /simulate o /optimize)
PATCH  /api/optimizations/{optimization_id}     # nueva iteración sobre un DRAFT existente
POST   /api/optimizations/{optimization_id}/consolidate
POST   /api/optimizations/{optimization_id}/revert-consolidation   # solo si nada posterior depende de él
GET    /api/optimizations/{optimization_id}/dependents            # meses futuros que se verían afectados
DELETE /api/optimizations/{optimization_id}     # soft-delete, solo sobre DRAFT
```

Puntos clave:
- `optimization_id` es un identificador **estable** de "la optimización de este centro+mes+año", no de cada `run_id` individual. Cada `PATCH` genera un nuevo `run_id` (para no perder el historial de iteraciones), pero todos cuelgan del mismo `optimization_id` lógico.
- Esto permite volver a `GET /api/optimizations/{id}` después de salir de la pantalla y seguir iterando sobre la misma entidad, resolviendo el "problema de edición" (1.1).
- Un `optimization_id` puede tener N `runs` (iteraciones); el frontend siempre trabaja sobre el último `run` no descartado, salvo que pida explícitamente el historial.

### A.2 Estados y máquina de transición

Añadir un campo de estado controlado (no un `VARCHAR` libre como el actual `process_status`):

```
DRAFT        -- optimización recién creada o recalculada desde cero, editable libremente
ITERATING    -- tiene al menos una iteración con restricciones manuales sobre el DRAFT inicial
CONSOLIDATED -- mes cerrado, inmutable, es "la realidad" para todo lo posterior
ARCHIVED     -- invalidado en cascada por edición de un mes anterior (soft-delete, no se borra físicamente)
```

Transiciones permitidas:

```
DRAFT ──iterar──> ITERATING ──iterar──> ITERATING
DRAFT|ITERATING ──consolidar──> CONSOLIDATED
DRAFT|ITERATING ──(editar mes anterior)──> ARCHIVED   [creado un nuevo DRAFT en su lugar]
CONSOLIDATED ──X──                                     (inmutable; ver A.3 para "what-if")
```

`CONSOLIDATED` nunca vuelve a `DRAFT` directamente: si hay que corregir un mes consolidado, es una operación administrativa explícita y auditada (ver A.2.1), no una transición normal del flujo de iteración.

#### A.2.1 Cascada de consolidación (enero-febrero-marzo)

Regla: **no puede existir un mes consolidado con un mes anterior (mismo centro) en estado distinto de `CONSOLIDATED`.**

Al llamar `POST /optimizations/{marzo_id}/consolidate`:
1. Backend calcula la cadena de meses anteriores del mismo centro que no estén ya `CONSOLIDATED`.
2. Si la lista no está vacía, responde `409 Conflict` con el detalle:
   ```json
   {
     "error": "cascade_consolidation_required",
     "requires_consolidation_of": [
       {"optimization_id": 101, "year": 2027, "month": 1, "status": "DRAFT"},
       {"optimization_id": 102, "year": 2027, "month": 2, "status": "ITERATING"}
     ]
   }
   ```
3. El frontend muestra el pop-up: *"Para consolidar marzo, enero y febrero deben consolidarse también. ¿Confirmas consolidar los 3 meses?"*
4. Si el usuario confirma, se llama `POST /optimizations/consolidate-batch` con la lista completa `[enero_id, febrero_id, marzo_id]`, y el backend lo ejecuta en **una única transacción** (todo o nada) para evitar quedar en un estado intermedio inconsistente si falla a mitad.
5. Si algún mes anterior no tiene ninguna optimización todavía (nunca se ejecutó), no se puede consolidar marzo: se debe crear y consolidar antes esos meses, o al menos declarar explícitamente "sin actividad" (para centros nuevos o meses sin donaciones).

#### A.2.2 Branching / "what-if" sobre meses consolidados

Un mes `CONSOLIDATED` nunca se modifica in-place. Para experimentar:

```
POST /api/optimizations/{consolidated_id}/branch
```

Esto crea una **copia** con:
- `status = DRAFT`
- `parent_run_id = consolidated_run_id` (trazabilidad: de qué consolidado partió)
- `is_scenario = true`
- Mismo `year`/`month`, pero marcado como escenario, **no** como el mes real: en los listados normales del frontend no debe aparecer mezclado con el histórico real salvo que el usuario entre explícitamente en "modo escenario".
- Los escenarios **no participan** en la cascada de consolidación ni en el cálculo de dependencias temporales de A.3: son un sandbox aislado, de solo lectura respecto al dato consolidado real.

### A.3 Dependencias temporales y soft-delete en cascada hacia adelante

Cuando el usuario edita (nueva iteración o recalcula desde cero) un mes en `DRAFT`/`ITERATING` que **no** está consolidado:

1. Backend consulta `GET /optimizations?center_id=X&after=year-month` para encontrar optimizaciones de meses posteriores del mismo centro que ya tengan al menos un `run`.
2. Si existen y **no están consolidados**: se muestran en el pop-up de advertencia (*"Vas a editar esta solución. Esto invalidará N meses posteriores: abril (DRAFT), mayo (ITERATING)"*), y si el usuario confirma, esos meses pasan a `ARCHIVED` (soft-delete: se conservan sus `runs` y ficheros para auditoría, pero se marcan `archived_at`, `archived_reason='cascade_invalidation'`, `archived_by_optimization_id=<el que originó la edición>`) y dejan de aparecer como "vigentes" en el calendario.
3. Si algún mes posterior **ya está `CONSOLIDATED`**: no se permite la edición retroactiva por esta vía. Se devuelve `409` explicando que hay meses consolidados por delante y que corregir el pasado consolidado es una operación administrativa distinta (fuera del flujo normal de iteración; requiere justificación y probablemente reconsolidar en cadena desde ese punto).
4. Todo el borrado en cascada es lógico (columna de estado + timestamp), nunca `DELETE` físico de filas, precisamente para permitir auditoría clínica de "qué se decidió y cuándo se invalidó".

---

## Bloque B — CSVs locales vs. base de datos

### B.1 Migración del almacenamiento de resultados y restricciones

Sustituir `constraints_json_path` (fichero en disco del contenedor) por una columna `constraints_json JSONB` directamente en `scheduler_input` (o una tabla `scheduler_run_constraints(run_id, constraints_json)` si se prefiere no engordar la tabla principal). Postgres en GCP (Cloud SQL) ya es el sistema con backups/durabilidad; no hay motivo para mantener un fichero intermedio que puede perderse en cualquier reinicio de contenedor.

Pasos de migración sin downtime:
1. Añadir la columna `constraints_json JSONB NULL` (migración aditiva, no rompe nada).
2. Backend escribe en **ambos sitios** (columna + fichero) durante una fase de transición corta, priorizando siempre la lectura desde la columna si existe.
3. Script de backfill: para los `constraints_json_path` existentes que todavía sean legibles (contenedor no reiniciado aún), leerlos y volcarlos a la columna nueva.
4. Una vez verificado que todo lo reciente está en columna, dejar de escribir en fichero y marcar `constraints_json_path` como deprecado (se puede conservar la columna vacía por trazabilidad histórica, sin borrar el esquema todavía).
5. Igual criterio para cualquier otro resultado "grande" que hoy viva solo en CSV de scheduler/DEVS (por ejemplo, el `optimization_result.csv` de `bloodtwin_scheduler_milp`): pasar a `scheduler_output`, que ya existe y ya modela día/turno/vehículo/valor — es decir, gran parte de este trabajo es "dejar de escribir el CSV intermedio y solo escribir en la tabla que el backend ya llena hoy tras leer ese mismo CSV".

Esto no rompe la funcionalidad interna de DEVS/OPL: ellos pueden seguir generando CSV como formato de intercambio *dentro* de su propio contenedor de cálculo (es su detalle de implementación), pero el backend deja de depender de que ese fichero siga vivo después de terminar la llamada HTTP: en el mismo request en que recibe la respuesta de `/optimize` o `/simulate`, persiste el resultado completo en Postgres y ya no vuelve a tocar el fichero.

### B.2 Histórico consolidado como fuente de verdad para simulación futura

Hoy `bloodtwin_scheduler_milp` lee `data/src/historical_*.csv`, un fichero único montado en el contenedor, para saber "hasta qué mes hay histórico real" y generar los meses intermedios que falten. Esto no escalará cuando el histórico real venga de meses `CONSOLIDATED` en base de datos.

Diseño propuesto:
- `bloodtwin_database_updater` (que **ya existe** y ya vuelca CSV/JSON subidos a GCS hacia Postgres mediante Cloud Function) se convierte en el punto único de entrada del histórico real mensual: cuando el equipo clínico sube el fichero de donaciones reales del mes que se acaba de cerrar, el updater lo escribe en las mismas tablas `scheduler_input`/`scheduler_output` (o una tabla `historical_collections` dedicada si conviene separar "dato real medido" de "salida de un algoritmo"), marcado con el mismo mecanismo de consolidación de A.2 (`status = CONSOLIDATED`, `source = 'real_data_import'`).
- El backend expone un endpoint de solo lectura consumido por scheduler/DEVS en lugar de un fichero:
  ```
  GET /api/history/base-state?center_id=&up_to=year-month
  ```
  que devuelve el estado base consolidado más reciente (stock, demanda agregada, etc.) necesario para arrancar la simulación del mes siguiente.
- `bloodtwin_scheduler_milp` y `bloodtwin_devs` dejan de tener un CSV histórico "hardcodeado" y en su lugar reciben ese estado base como parte del payload que ya les manda el backend (`generalInfo`, `scheduling`, y ahora también `baseState`), igual que ya reciben `predictedDemand` hoy. Es una extensión del contrato existente, no un rediseño de esos servicios.
- Mientras conviven ambos mundos (migración progresiva), el backend puede seguir aceptando el CSV como *fallback* si `baseState` no está disponible para un centro que aún no tiene histórico consolidado en BD, pero registrando un warning explícito para forzar la migración completa.

### B.3 Orquestación y rendimiento del bucle de simulación DEVS

Dependencia identificada: predicción de demanda → modelo de stock → scheduler, y todo depende del histórico consolidado.

Recomendación: **cargar en memoria al inicio de la simulación de un mes, no on-the-fly por evento DEVS.**

Motivos:
- El propio `devs_server.py` ya es un proceso de larga vida con un `Coordinator` construido una vez (`DevsEngine.__init__`) y un `threading.Lock()` que serializa simulaciones; introducir una consulta a Postgres por cada evento interno del modelo DEVS multiplicaría la latencia de red dentro de un bucle que hoy es puramente en memoria/CPU.
- El volumen de datos de "estado base para un mes" (stock inicial, demanda histórica agregada, calendario de festivos/vísperas) es pequeño comparado con el número de eventos internos de la simulación día a día.
- Patrón propuesto: el backend resuelve `GET /api/history/base-state` **antes** de invocar `/simulate` u `/optimize`, y lo incluye en el payload HTTP como un bloque de datos ya materializado (arrays/diccionarios), igual que hoy ya hace con `predicted_demand`. DEVS y el optimizador consultan ese bloque en memoria durante toda la corrida; no abren conexión a base de datos.
- Si en el futuro el volumen de histórico crece mucho (multi-año, muchos centros a la vez), considerar cachear `base-state` por `(center_id, year, month)` con invalidación al consolidar/archivar, en vez de recalcularlo en cada llamada.

Cuello de botella a vigilar: el `threading.Lock()` global de `DevsEngine` implica que **todas las simulaciones de todos los centros se serializan** en el mismo proceso. Si el volumen de optimizaciones concurrentes crece (varios centros consolidando el mismo mes a la vez), esto se convierte en cuello de botella independientemente de dónde viva el histórico. Vale la pena decidir explícitamente si se acepta esa serialización (probablemente sí, dado el volumen clínico actual) o si se pasa a un pool de workers por centro.

---

## Huecos no mencionados en el enunciado original

1. **Multi-centro y multi-región**: el modelo de estados/consolidación debe llevar siempre `center_id` (y probablemente `region_id`) en la clave de unicidad de `MonthlyOptimization`, no solo `year+month`. El código actual (`scheduler_input.center_name`, `OptimizeGeneralInfo.centerId`) ya lo soporta parcialmente pero conviene dejarlo explícito en el nuevo diseño para no acoplar accidentalmente calendarios de centros distintos.
2. **Auditoría de quién consolida/edita**: dado el uso clínico, cada transición de estado (`consolidate`, `archive`, `branch`) debería registrar `user_id` y `timestamp` en una tabla de auditoría append-only, no solo el estado final. Esto es fácil de olvidar y crítico si hay una revisión clínica o legal posterior.
3. **Idempotencia de la Cloud Function del Database Updater**: si el fichero real de un mes se sube dos veces (reintento, corrección de un typo), hay que decidir si sobrescribe el consolidado (prohibido según A.2 salvo proceso administrativo) o crea una nueva versión "real" a revisar manualmente antes de reemplazar el consolidado.
4. **Notificación de invalidación**: cuando se archivan meses futuros en cascada (A.3), alguien responsable de esos meses debería recibir una notificación (no solo el usuario que hizo el cambio), especialmente si otro miembro del equipo era quien estaba iterando ese mes.
5. **Backups/retención de los ficheros JSON ya existentes**: antes de dejar de escribir en `CONSTRAINTS_BASE_PATH`, hacer un volcado único de todo lo que sea recuperable a la nueva columna, para no perder historial de iteraciones antiguas ya hechas en producción.
6. **Definición de "sin actividad" para el requisito de cascada (A.2.1)**: centros nuevos o meses sin donaciones necesitan una forma explícita de marcar "no aplica" sin forzar una consolidación vacía artificial.
7. **Contrato de error único**: hoy cada servicio (`devs_server.py`, `api.py` del MILP, backend) tiene su propio formato de error ad-hoc; al introducir estados y cascadas con más casuística de rechazo (`409`), conviene unificar el formato de error entre los tres servicios para que el frontend no tenga que interpretar 3 formatos distintos.
8. **Tests de regresión de la lógica de borrado/archivado**: dado que hoy `save_simulation_run` ya borra filas physically en el caso "sin constraints", cualquier cambio a soft-delete debe venir acompañado de tests que verifiquen que ya no hay `DELETE` físico de `dispatcher_processes`/`scheduler_input`/`scheduler_output` fuera de los casos explícitamente decididos como purga administrativa.

---

## Bloque C — Hoja de ruta priorizada

El orden prioriza: (a) parar la pérdida de datos activa hoy, (b) introducir estado sin romper el flujo actual, (c) construir consolidación/cascada, (d) migrar histórico a BD, (e) limpieza final. Cada fase es desplegable de forma independiente y no bloquea el uso clínico mientras se desarrolla.

### Fase 0 — Mitigación inmediata (días)
- [ ] Montar un volumen persistente para `CONSTRAINTS_BASE_PATH` en el contenedor del backend (mitiga la pérdida de datos ya, sin cambiar código).
- [ ] Añadir alerta/log claro cuando `get_simulation_history`/recuperación de un run falle por fichero ausente, para detectar el problema en vez de un 500 genérico.

### Fase 1 — Modelo de datos y estado (1–2 sprints)
- [ ] Migración aditiva: columna de estado controlado en `dispatcher_processes` (o tabla nueva `optimization_state`), columna `constraints_json JSONB`, columnas de auditoría (`archived_at`, `archived_reason`, `archived_by_optimization_id`, `is_scenario`, `parent_run_id`).
- [ ] Introducir el concepto de `optimization_id` estable que agrupe los `run_id` de un mismo center+year+month.
- [ ] Backend escribe en columna nueva y en fichero en paralelo (fase de transición, ver B.1).
- [ ] Tests de la máquina de estados (transiciones válidas/ inválidas) antes de exponerla en la API.

### Fase 2 — API de optimizaciones (1–2 sprints)
- [ ] Endpoints de A.1 (`GET/POST/PATCH/DELETE /api/optimizations`, `dependents`).
- [ ] Lógica de dependencias temporales hacia adelante (A.3) con soft-delete/archivado.
- [ ] Frontend: pop-up de advertencia de invalidación futura y pantalla para recuperar/seguir iterando un `optimization_id` existente (resuelve 1.1 directamente).

### Fase 3 — Consolidación y cascada (1–2 sprints)
- [ ] Endpoint de consolidación individual + `consolidate-batch` transaccional (A.2.1).
- [ ] Endpoint de branching/what-if (A.2.2) y marcado `is_scenario` en listados del frontend.
- [ ] Frontend: botón "Consolidar mes", pop-up de cascada, vista separada de escenarios what-if.
- [ ] Auditoría append-only de transiciones (hueco 2).

### Fase 4 — Histórico consolidado en base de datos (2–3 sprints, el más grande)
- [ ] Extender `bloodtwin_database_updater` (o crear un flujo hermano reutilizando su patrón de Cloud Function) para volcar el histórico real de scheduler/donaciones a Postgres marcado como `CONSOLIDATED`/`source='real_data_import'`.
- [ ] Endpoint `GET /api/history/base-state` en el backend.
- [ ] `bloodtwin_scheduler_milp` y `bloodtwin_devs` reciben `baseState` en el payload en lugar de leer `historical_*.csv`; mantener el CSV como fallback con warning durante la transición (B.2).
- [ ] Backfill del histórico ya existente en CSV hacia la base de datos, con validación manual del primer volcado antes de confiar en él para producción.

### Fase 5 — Limpieza y cierre (1 sprint)
- [ ] Dejar de escribir `constraints_json_path` en disco; deprecar la columna manteniendo lectura por compatibilidad.
- [ ] Retirar la dependencia de `historical_*.csv` una vez validado que todos los centros activos tienen histórico en BD.
- [ ] Unificar formato de error entre los tres servicios (hueco 7).
- [ ] Documentar el modelo de estados y el flujo de consolidación en el README de `bloodtwin_backend` y `bloodtwin_database`.

---

## Siguiente paso sugerido

Cuando se apruebe este documento, dividir cada casilla de la Sección "Bloque C" en una issue del repositorio correspondiente (`bloodtwin_backend` para Fases 1–3 y 5, `bloodtwin_database` para las migraciones de esquema, `bloodtwin_database_updater` y `bloodtwin_scheduler_milp`/`bloodtwin_devs` para la Fase 4), enlazando cada issue a este archivo como contexto de diseño.
