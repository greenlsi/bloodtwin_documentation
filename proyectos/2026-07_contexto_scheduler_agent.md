# Contexto histórico del agente scheduler

Resumen de la transcripción `conversacion_bloodtwin_scheduler_agent.txt`, incorporada
al repositorio padre por **Alex from Asusito** el 20 de julio de 2026, en el commit
`d1ea78098eb642a08fe028ec20b64836b6a80ba1` ("Checkpoint in the UI."). La autoría
del commit identifica a quien publicó el archivo, no necesariamente al interlocutor
del chat. La transcripción original permanece recuperable en ese commit del padre.

## Objetivo de la conversación

Adaptar el prototipo Colab `opl_modifier_v6.py` a la arquitectura de
`bloodtwin_scheduler_agent`, preservando la integración con backend, frontend y
optimizador MILP. Se mencionó la rama `issue/adapt-notebooks-to-scripts`.

## Criterios discutidos

- Publicar primero los commits de cada submódulo y después sus punteros en el padre:
  un puntero a un commit no publicado no se puede reproducir al clonar.
- Separar interpretación de restricciones y modificación/validación del modelo OPL;
  no mezclar sus contratos HTTP.
- Mantener el flujo de restricciones propuestas y confirmadas entre frontend,
  backend y agente.
- Ubicar los modelos OPL en el entorno del scheduler MILP, no duplicarlos en el
  agente; aislar los artefactos de cada ejecución.
- Sustituir rutas personales de Drive por configuración y contratos de datos
  explícitos. No depender de un CSV de calendario como persistencia de la interfaz.
- Configurar modelo LLM y credenciales mediante el entorno; no incluir claves en
  código, imágenes Docker ni ejemplos versionados.
- Reutilizar la declaración de dependencias del servicio y añadir pruebas del
  contrato, validación OPL y errores de integración.

## Alcance de esta nota

Estos puntos resumen decisiones y propuestas de una conversación histórica. No
certifican que estén implementados ni sustituyen al código, los README actuales o
las issues de cada repositorio. Antes de retomar una tarea, comprobar su estado en
`bloodtwin_scheduler_agent` y `bloodtwin_scheduler_milp`.