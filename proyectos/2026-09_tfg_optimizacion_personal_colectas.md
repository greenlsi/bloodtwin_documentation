# TFG: optimización del calendario de personal para colectas

**Estado:** propuesta inicial para la reunión de arranque.
**Contexto:** este trabajo se sitúa *después* del scheduler de colectas. El scheduler decide
**dónde y cuándo** se pone cada unidad móvil; este TFG decide **quién va** a cada una.

---

## 1. Qué existe ya (verificado en el código)

### 1.1 El cálculo de personal necesario **sí está hecho**

Está en [`hemoglobulopt/control_variables.py`](https://github.com/greenlsi/bloodtwin_scheduler_milp)
y se aplica en `hemoglobulopt/util.py`. Es una tabla escalonada que, a partir de la previsión
de donantes de una colecta (`FORECAST`), devuelve cuánta gente hace falta:

| Previsión de donantes (≤) | Enfermeros | Administrativos | Médicos | Conductores |
| --- | --- | --- | --- | --- |
| 45 | 2 | 1 | 1 | 1 |
| 65 | 3 | 1 | 1 | 1 |
| 79 | 4 | 1 | 1 | 1 |
| 99 | 5 | 1 | 1 | 1 |
| 114 | 6 | 2 | 1 | 1 |
| 124 | 7 | 2 | 1 | 1 |
| 139 | 8 | 2 | 1 | 1 |
| 149 | 9 | 2 | 1 | 1 |
| 169 | 10 | 2 | 1 | 1 |
| más | 11 | 2 | 1 | 1 |

Existe también `PERSONNEL_COSTS`, con el coste por persona y tipo de módulo (A, B, C, D).

**Matices importantes que hay que decirle a la alumna desde el principio:**

- Los datos son **de 2018** y no se han revisado desde entonces.
- Hoy se usan **solo para calcular coste**, como cálculo posterior. No son una variable de
  decisión ni asignan personas concretas.
- Si una colecta no tiene `FORECAST`, el código lo estima como `DONACIONES / 0.85`. Hay un
  `TODO` explícito en el código reconociendo que la columna de previsión está incompleta y
  se rellenó a mano.
- El número de médicos es siempre 1 y el de conductores siempre 1, independientemente del
  tamaño. Conviene confirmar con el centro si eso sigue siendo así.

**Conclusión:** la entrada del TFG está disponible. Para cada colecta planificada se sabe el
día, el turno, el punto y cuántas personas de cada categoría hacen falta. No hay que
calcularlo de nuevo, pero sí **validar la tabla con el centro** antes de construir nada
encima.

### 1.2 Lo que el scheduler entrega

La salida del scheduler (tabla `scheduler_output`) da, por cada colecta: día, punto de
extracción (`ep_id`), turno (`shift`), vehículo y donaciones previstas. Es decir, el TFG
recibe una lista de "puestos a cubrir" ya cerrada.

---

## 2. El hueco real: no existe información de quién fue a cada colecta

Esto es lo más importante de la reunión, porque condiciona todo el trabajo.

**La idea de inferir la disponibilidad a partir del histórico no se puede aplicar tal cual.**
El histórico (`historical_2007-2023.csv`, 2007–2023) tiene estas columnas:

```
DATE, YEAR, MONTH, DAY, WEEK, WEEK_ABSOLUTE, MONTH_ABSOLUTE, DAY_OF_WEEK, SEASON,
ID, ID_NO_LETTER, COLLECTION, MODE, BLOCKED, VALIDATED, FORECAST, DONORS,
DONATIONS, DONATIONS_IDEAL, TUBO, RECHAZADOS, RECHAZOS CALCULADOS, IR EN EL PUNTO DE COLECTA
```

No hay **ninguna columna de persona** ni de turno. El histórico registra *colectas*, no
*asignaciones de personal*. Por tanto no se puede aplicar el razonamiento "fue este día,
luego ese día estaba disponible": no consta quién fue.

En la base de datos existe una tabla `medical_team` con `full_name`, `email`,
`specialty_type`, `seniority_years`, `collections_count`, `rejection_rate` y
`absences_count`, pero hoy está poblada con **datos de ejemplo inventados** y, aunque
tuviera datos reales, son agregados por persona: no dicen qué día fue cada uno.

### Opciones, en orden de preferencia

**A. Pedir los datos reales al centro (lo que hay que intentar primero).**
Lo que haría falta es el cuadrante histórico: por cada colecta pasada, qué personas fueron y
en qué turno. Aunque sean dos o tres años, o incluso unos meses. Con eso la inferencia de
disponibilidad que planteas sí es viable y el TFG gana muchísimo valor. Conviene preguntar
también por vacaciones, permisos, contratos a tiempo parcial y convenio (descansos mínimos,
máximo de fines de semana seguidos, etc.).

**B. Generar un conjunto de datos sintético pero realista, documentado como tal.**
Es perfectamente defendible en un TFG siempre que se explique el procedimiento y que el
modelo esté preparado para consumir datos reales sin cambios. Se construye una plantilla de
plantilla ficticia (por ejemplo 40 enfermeros, 8 administrativos, 6 médicos, 10 conductores)
y se les generan vacaciones, contratos y preferencias con una distribución razonable.

Aquí sí se puede usar el histórico, pero para otra cosa: **la carga de trabajo real**.
Del histórico se saca cuántas colectas hubo cada día y de qué tamaño, y por tanto cuánta
gente hizo falta cada día de los últimos años. Eso permite dimensionar la plantilla ficticia
para que el problema sea realista y no trivial: ni sobra gente ni es infactible.

**C. Tratar la disponibilidad como escenarios paramétricos.**
En lugar de un cuadrante concreto, se definen escenarios (verano con 30 % de plantilla de
vacaciones, invierno con picos de absentismo, etc.) y se estudia cómo responde el modelo.
Es una vía legítima y da una sección de resultados interesante.

**Recomendación:** empezar por B para no bloquear a la alumna, pedir A en paralelo, y dejar
el modelo preparado para que el origen de los datos sea intercambiable. Si A llega, se
sustituye el fichero y se reejecuta.

---

## 3. Objetivos

### Objetivo principal

Desarrollar un modelo de optimización en OPL/CPLEX que asigne el personal disponible a las
colectas planificadas por el scheduler, minimizando coste y respetando las restricciones
laborales, de disponibilidad y de cualificación.

### Objetivos secundarios

1. Caracterizar la demanda de personal a partir del histórico de colectas 2007–2023.
2. Construir y documentar el conjunto de datos de plantilla y disponibilidad.
3. Validar y, si procede, actualizar la tabla de necesidades de personal de 2018.
4. Definir el formato de intercambio entre el scheduler y el nuevo optimizador.
5. Empaquetar el modelo como servicio en Docker, siguiendo el patrón del scheduler actual.
6. Evaluar el modelo: calidad de la solución, tiempo de cálculo y comportamiento al crecer
   el tamaño del problema.

### Objetivos opcionales (si da tiempo)

7. Integrar el servicio en el simulador DEVS.
8. Equilibrio de carga entre personas (que no siempre vayan los mismos a los sitios malos).
9. Robustez ante bajas de última hora: replanificación con mínimo cambio.

---

## 4. Repositorios

### Lo que propongo crear: **uno solo**

```
bloodtwin_staffing_milp
```

Con la misma arquitectura que `bloodtwin_scheduler_milp`, que ya está probada:

```
bloodtwin_staffing_milp/
├── opl_model/              modelo .mod y datos .dat
├── staffing/               envoltorio Python: prepara datos, invoca OPL, lee resultados
│   ├── control_variables.py    parámetros del problema
│   ├── generate_opl_data.py    construye el .dat desde la entrada
│   └── util.py
├── data/
│   ├── src/                datos de entrada (plantilla, disponibilidad, calendario)
│   └── opl_data/           generados y resultados
├── tests/
├── api.py                  endpoint HTTP, como el del scheduler
├── Dockerfile
└── README.md
```

### Por qué no dos repositorios

La parte de Docker y de integración con DEVS **no necesita repositorio propio**. En este
proyecto cada servicio vive en su repositorio con su propio `Dockerfile` y su `api.py`, y el
repositorio padre `bloodtwin` los orquesta con `docker-compose.yml`. Eso es exactamente lo
que hace `bloodtwin_scheduler_milp` hoy.

Crear ahora un repositorio "para la integración" produciría un repositorio vacío durante
meses. Ya tenemos el precedente de `bloodtwin_device_fw`, que estuvo con la rama principal
vacía mientras el trabajo real vivía en otro sitio, y eso genera confusión. La integración
con DEVS es un cambio de contrato en repositorios que ya existen, no un repositorio nuevo.

Cuando llegue el momento, el trabajo consiste en: añadir el servicio al `docker-compose.yml`
del padre, y añadir en `bloodtwin_devs` el modelo DEVS que lo invoque.

---

## 5. Qué contarle sobre DEVS en la primera reunión

**Mencionarlo, sin entrar en detalle.** Una frase del tipo: *"esto acabará siendo un servicio
dentro de un simulador de eventos discretos que ya existe; no te preocupes ahora, pero tenlo
en cuenta para no diseñarlo como un script que solo funciona en tu portátil"*.

Contarle DEVS entero en la primera reunión es contraproducente: es un formalismo que requiere
tiempo y no lo necesita para modelar el problema. Pero si no se le dice nada, acabará con un
cuaderno de Jupyter que lee rutas absolutas de su disco, y la integración será dolorosa.

Lo que sí hay que exigirle desde el día uno, y es suficiente:

- Entrada y salida en ficheros con formato definido, nunca rutas fijas del disco.
- Los parámetros en un fichero de configuración, no incrustados en el código.
- Que el modelo se pueda invocar desde un script sin abrir el IDE de CPLEX.

Con eso, meterlo en Docker y en DEVS más adelante es trivial.

---

## 6. Lecturas recomendadas

### Del proyecto (en `bloodtwin_documentation/tfts/`)

Por orden de utilidad para ella:

1. **`2023-eduardoAbreu-TFM.pdf`** — es el más relevante. Es la versión actual del modelo OPL
   del scheduler, el que está en producción. Debe entender la estructura del `.mod`, cómo se
   generan los `.dat` desde Python y cómo se leen los resultados.
2. **`2019-eduardoFernandez-TFM.pdf`** — el modelo MILP original. Útil para entender por qué
   el problema se formuló así y qué se descartó.
3. **`2020-nachoUranga-TFM.pdf`** — solo si se llega a la parte DEVS. No es prioritario ahora.

### Sobre el problema (nurse rostering)

Su problema es un **Nurse Rostering Problem** con una particularidad: los turnos no son en un
hospital fijo, sino en colectas móviles con ubicación y tamaño variables. Referencias
clásicas y verificables:

- Burke, De Causmaecker, Vanden Berghe, Van Landeghem (2004), *The State of the Art of Nurse
  Rostering*, Journal of Scheduling. Es el punto de partida canónico.
- Ernst, Jiang, Krishnamoorthy, Sier (2004), *Staff scheduling and rostering: A review of
  applications, methods and models*, European Journal of Operational Research.
- Van den Bergh, Beliën, De Bruecker, Demeulemeester, De Boeck (2013), *Personnel scheduling:
  A literature review*, European Journal of Operational Research. El más reciente de los tres
  y el más completo.

Para datos y comparación: las **International Nurse Rostering Competitions** (INRC-I, 2010 e
INRC-II, 2015) publicaron instancias y formatos de referencia. Vale la pena que los mire
aunque solo sea para ver cómo se modelan las restricciones de convenio.

### Sobre OPL y CPLEX

- Producto y descarga: IBM ILOG CPLEX Optimization Studio
  (<https://www.ibm.com/products/ilog-cplex-optimization-studio>).
- **Licencia académica:** se obtiene gratis a través del *IBM Academic Initiative* / *IBM
  SkillsBuild for Academia*, registrándose con el correo institucional de la UPM. La versión
  gratuita sin licencia académica tiene un límite de 1000 variables y 1000 restricciones, que
  se queda corto enseguida: conviene que tramite la licencia académica **en la primera
  semana**, porque a veces tarda en concederse.
- La documentación oficial incluye el *OPL Language Reference Manual* y el *OPL Language
  User's Manual*, y la instalación trae ejemplos resueltos, entre ellos varios de asignación
  de personal.

Una nota práctica: que trabaje desde el principio invocando el modelo con `oplrun` desde línea
de comandos, no solo desde el IDE. Es lo que hará falta para automatizarlo.

---

## 7. Plan por fases

### Fase 0 — Preparación (semanas 1–2)

- [ ] Solicitar la licencia académica de CPLEX. **Hacerlo el primer día.**
- [ ] Instalar CPLEX Optimization Studio y ejecutar un ejemplo de los incluidos.
- [ ] Leer el TFM de Eduardo Abreu y ejecutar el scheduler actual para ver entradas y salidas.
- [ ] Dar acceso de lectura a `bloodtwin_scheduler_milp`.
- [ ] Reproducir un modelo sencillo de nurse rostering de la literatura, con datos de juguete.

### Fase 1 — Datos (semanas 3–5)

- [ ] Analizar el histórico: colectas por día, tamaño, estacionalidad.
- [ ] Traducir eso a demanda diaria de personal usando la tabla `PERSONNEL_NEEDS`.
- [ ] Pedir al centro el cuadrante histórico real y los criterios de convenio.
- [ ] Construir el conjunto de datos de plantilla y disponibilidad, documentando cada
      supuesto que se invente.
- [ ] Definir el formato de intercambio con el scheduler.

### Fase 2 — Modelo base (semanas 6–10)

- [ ] Formular el modelo: variables, función objetivo y restricciones duras.
- [ ] Implementarlo en OPL con datos reducidos, por ejemplo un mes y un centro.
- [ ] Validar que las soluciones son factibles y tienen sentido.
- [ ] Añadir las restricciones blandas (preferencias, equilibrio de carga).

### Fase 3 — Escalado y evaluación (semanas 11–14)

- [ ] Ejecutar sobre un año completo.
- [ ] Medir tiempos y calidad; estudiar dónde empieza a costar.
- [ ] Comparar contra una heurística sencilla o contra la asignación manual, si se consigue.

### Fase 4 — Integración (semanas 15–17)

- [ ] Envoltorio Python que genere el `.dat` y lea los resultados.
- [ ] Endpoint HTTP siguiendo el patrón de `api.py` del scheduler.
- [ ] `Dockerfile` y alta del servicio en el `docker-compose.yml` del padre.
- [ ] Si da tiempo: modelo DEVS que lo invoque.

### Fase 5 — Redacción (en paralelo desde la fase 1)

- [ ] Que escriba desde el principio, no al final. El estado del arte puede redactarlo ya
      durante la fase 0.

---

## 8. Primeros pasos concretos para la reunión

Lo que debería salir de la reunión, por orden:

1. **Tramitar hoy la licencia académica de CPLEX.** Es lo único que puede bloquearla por
   motivos ajenos a ella.
2. Leer el TFM de Eduardo Abreu y venir con preguntas a la siguiente reunión.
3. Acceso de lectura a `bloodtwin_scheduler_milp` y al repositorio nuevo cuando se cree.
4. Reproducir un nurse rostering de juguete en OPL, para soltarse con el lenguaje.
5. Dejar claro desde el principio que el histórico **no tiene datos de personal**, para que
   no pierda semanas buscándolos, y que parte del trabajo será construir ese conjunto de
   datos de forma justificada.

### Preguntas abiertas que hay que resolver con el centro

- ¿Existe el cuadrante histórico de personal? ¿En qué formato?
- ¿Sigue siendo válida la tabla de necesidades de 2018?
- ¿Qué convenio aplica: descansos mínimos, máximo de días seguidos, fines de semana?
- ¿Hay personal fijo por zona geográfica o todos pueden ir a cualquier punto?
- ¿Los conductores pueden hacer otras tareas o son exclusivos?
- ¿Cuánta gente hay realmente en plantilla, por categoría?
