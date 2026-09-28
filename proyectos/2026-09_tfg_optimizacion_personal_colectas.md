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

- Los datos son **de 2018** y el centro confirma que **siguen siendo válidos**.
- Hoy se usan **solo para calcular coste**, como cálculo posterior. No son una variable de
  decisión ni asignan personas concretas.
- Si una colecta no tiene `FORECAST`, el código lo estima como `DONACIONES / 0.85`. Hay un
  `TODO` explícito en el código reconociendo que la columna de previsión está incompleta y
  se rellenó a mano. Ese error se arrastra al dimensionado del personal.
- El número de médicos es siempre 1 y el de conductores siempre 1, independientemente del
  tamaño de la colecta.

El **coste** depende del módulo del punto de colecta (`MODULE` en `structural.csv`, valores
`A` a `D`). Un mismo perfil cuesta un 39 % más en un módulo `D` que en uno `A`. La tabla
completa de costes está en el README de `bloodtwin_staffing_milp`.

**Conclusión:** la entrada del TFG está disponible. Para cada colecta planificada se sabe el
día, el turno, el punto, el módulo, el tipo de colecta y cuántas personas de cada categoría
hacen falta.

### 1.2 Lo que el scheduler entrega

La salida del scheduler (tabla `scheduler_output`) da, por cada colecta: día, punto de
extracción (`ep_id`), turno (`shift`), vehículo y donaciones previstas. Es decir, el TFG
recibe una lista de "puestos a cubrir" ya cerrada.

---

## 2. Situación de los datos de personal

**Los datos de personal existen.** El centro dispone del histórico de quién fue a cada
colecta y cuándo, y de quién no estaba disponible por vacaciones o por estar bloqueado. No
viven en los repositorios del proyecto, sino en tablas propias del centro, y son datos
laborales identificables: no deben acabar en ningún repositorio de código.

Eso hace viable desde el principio el modelo base, sin necesidad de inventar la plantilla.

### Lo que sí falta

**Preferencias personales.** No consta si alguien prefiere mañana o tarde, ni qué destinos
prefiere, ni con quién trabaja mejor. Es justo lo que daría más riqueza al modelo, porque son
las restricciones blandas que convierten un problema de cobertura en un problema de
satisfacción del personal.

Aquí sí cabe la inferencia a partir del histórico, con cautela: ver la **proporción de turnos
de mañana frente a los de tarde** de cada persona a lo largo de los años. Si alguien tiene un
reparto claramente desequilibrado de forma sostenida, es un indicio razonable de preferencia
o de disponibilidad estructural.

El riesgo es confundir preferencia con imposición: puede que esa persona hiciera siempre
mañanas porque se lo asignaban, no porque lo prefiriera. Hay dos formas de mitigarlo:

- Contrastar contra la mezcla de turnos disponible. Si en su zona el 80 % de las colectas
  eran de mañana, hacer el 80 % de mañanas no dice nada; hacer el 100 % sí.
- Validar la inferencia con el centro o con una encuesta breve a la plantilla. Con veinte
  respuestas se puede comprobar si la inferencia acierta, y eso da una sección de validación
  muy presentable en la memoria.

Mi recomendación: tratar las preferencias inferidas como **restricción blanda con peso bajo**
y marcarlas explícitamente como inferidas, no como dato. Y dejar el modelo preparado para
sustituirlas por preferencias declaradas si algún día se recogen.

### Tasas de rechazo por perfil médico

El trabajo previo del grupo sobre tasas de rechazo según la especialidad del médico, el tipo
de colecta y si está acompañado en la entrevista es una vía natural de extensión. Los datos
necesarios existen en parte: `structural.csv` ya tiene el tipo de colecta de cada punto y la
tabla `medical_team` del esquema contempla `specialty_type` y `rejection_rate`.

No entraría en la primera versión. Cuando el modelo base funcione, se incorpora como coste o
restricción: asignar médicos a los tipos de colecta donde su tasa de rechazo es menor, o
forzar acompañamiento donde más se nota. Es un buen objetivo secundario porque diferencia el
TFG de un nurse rostering estándar.

---

## 3. Objetivos

### Objetivo principal

Desarrollar un modelo de optimización en OPL/CPLEX que asigne el personal disponible a las
colectas planificadas por el scheduler, minimizando coste y respetando las restricciones
laborales, de disponibilidad y de cualificación.

### Objetivos secundarios

1. Caracterizar la demanda de personal a partir del histórico de colectas 2007–2023.
2. Preparar y anonimizar los datos de plantilla y disponibilidad del centro.
3. Inferir preferencias de turno a partir del histórico y validarlas.
4. Definir el formato de intercambio entre el scheduler y el nuevo optimizador.
5. Empaquetar el modelo como servicio en Docker, siguiendo el patrón del scheduler actual.
6. Evaluar el modelo: calidad de la solución, tiempo de cálculo y comportamiento al crecer
   el tamaño del problema.

### Objetivos opcionales (si da tiempo)

7. Incorporar las tasas de rechazo por especialidad médica y tipo de colecta.
8. Integrar el servicio en el simulador DEVS.
9. Equilibrio de carga entre personas.
10. Robustez ante bajas de última hora: replanificación con mínimo cambio.

---

## 4. Repositorios

### Lo que propongo crear: **uno solo**

```
bloodtwin_staffing_milp
```

**Ya está creado**, privado, y dado de alta como submódulo del repositorio padre. Su README
recoge los datos de entrada, la tabla de necesidades de personal, los costes por módulo y el
aviso sobre la previsión de donantes.

Tiene la misma arquitectura que `bloodtwin_scheduler_milp`, que ya está probada:

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
- [ ] Preparar y anonimizar los datos de plantilla, asistencia y ausencias del centro.
- [ ] Recabar los criterios de convenio: descansos, máximos, fines de semana.
- [ ] Inferir preferencias de turno del histórico y contrastarlas con la mezcla disponible.
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
3. Acceso de lectura a `bloodtwin_scheduler_milp` y a `bloodtwin_staffing_milp`.
4. Reproducir un nurse rostering de juguete en OPL, para soltarse con el lenguaje.
5. Acordar cómo se le entregan los datos de personal del centro, ya anonimizados, y dejar
   claro que no pueden subirse a ningún repositorio.

### Preguntas abiertas que hay que resolver con el centro

- ¿Qué convenio aplica: descansos mínimos, máximo de días seguidos, fines de semana?
- ¿Hay personal fijo por zona geográfica o todos pueden ir a cualquier punto?
- ¿Los conductores pueden hacer otras tareas o son exclusivos?
- ¿Se han recogido alguna vez preferencias declaradas de turno o destino?
- ¿Se puede hacer una encuesta breve a la plantilla para validar las preferencias inferidas?
