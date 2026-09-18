# Análisis del Módulo Actuator

## 1. Análisis del funcionamiento del código fuente

El conjunto de estos tres archivos define el comportamiento del módulo Actuador (Actuator), que es la etapa de salida del sistema. Su responsabilidad es traducir las órdenes lógicas en cambios eléctricos reales sobre el hardware (en este caso, un LED que emula a los actuadores físicos).

*   **`task_actuator_attribute.h`:** Define las estructuras de datos y los tipos enumerados fundamentales. Aquí se declaran los eventos posibles (`EV_LED_IDLE`, `EV_LED_ACTIVE`), los estados de la máquina (`ST_LED_IDLE`, `ST_LED_ACTIVE`) y la estructura de configuración `task_actuator_cfg_t` (que asocia un actuador con su pin y puerto GPIO específico).
*   **`task_actuator.c`:** Contiene la lógica principal del módulo. Incluye la inicialización del hardware (`task_actuator_init`), la rutina de actualización periódica (`task_actuator_update`) y la máquina de estados finitos (`task_actuator_statechart`) que decide qué hacer en base al estado actual y las variables de control.
*   **`task_actuator_interface.c`:** Proporciona la función `put_event_task_actuator()`, que actúa como el "buzón" o punto de entrada para que otros módulos (como el System) puedan enviarle comandos asincrónicos a este actuador.

## 2. Evolución de las variables del Actuador (init y update)

Al ejecutarse `task_actuator_init()` y en las posteriores llamadas a `task_actuator_update()`, las variables de `task_actuator_dta_list` evolucionan de la siguiente manera:

*   **`index`:** Es una variable local utilizada como iterador en los bucles `for`. Como en este código base solo hay configurado un actuador en `task_actuator_cfg_list` (el `ID_LED_A`), `ACTUATOR_DTA_QTY` vale 1. Por ende, `index` toma el valor `0` en la primera y única iteración del bucle, procesando los datos del primer LED.
*   **`task_actuator_dta_list[index].tick` (Unidad: Milisegundos o ciclos):** No se inicializa explícitamente en la función `_init` y no sufre modificaciones de incremento/decremento dentro de la máquina de estados. Solo se le asigna el valor `DEL_LED_MIN` (0) como resguardo si la máquina de estados cae en el caso `default`.
*   **`task_actuator_dta_list[index].state`:**
    *   Arranca inicializado en `ST_LED_IDLE` durante `task_actuator_init()`.
    *   Durante el `update`, si recibe la orden de activación, transita al estado `ST_LED_ACTIVE`.
    *   Si recibe la orden de reposo, vuelve a `ST_LED_IDLE`.
*   **`task_actuator_dta_list[index].event`:** Inicia en `EV_LED_IDLE` en el arranque. A partir de ahí, su valor cambia dinámicamente cuando el sistema central le inyecta nuevos eventos mediante la interfaz.
*   **`task_actuator_dta_list[index].flag`:** Se inicializa en `false`. Esta bandera sirve para avisarle a la máquina de estados que llegó un evento nuevo que no ha sido procesado. Se pondrá en `true` externamente cuando llegue una orden y se volverá a poner en `false` apenas la máquina de estados consuma y ejecute dicha orden.

## 3. Comportamiento de `task_actuator_statechart(uint32_t index)`

Esta función es el núcleo lógico que evalúa qué hacer con el pin GPIO:

1.  **Lectura de punteros:** Extrae la configuración (pin, puerto) y los datos de estado actuales del actuador indicado por `index`.
2.  **Evaluación (switch-case):**
    *   **Si está en `ST_LED_IDLE`:** Evalúa si la variable `flag` es `true` y si el evento guardado es `EV_LED_ACTIVE`. Si se cumplen ambas, baja la bandera (`flag = false`), escribe en el pin GPIO para encender el LED (`led_on`) y cambia el estado interno a `ST_LED_ACTIVE`.
    *   **Si está en `ST_LED_ACTIVE`:** Evalúa si la `flag` es `true` y el evento es `EV_LED_IDLE`. Si es así, baja la bandera, apaga el LED mediante el GPIO (`led_off`) y retorna al estado de reposo `ST_LED_IDLE`.
    *   **Caso `default`:** Sirve como mecanismo de seguridad (fail-safe). Si por algún error de memoria o corrupción el estado toma un valor inválido, el sistema fuerza todas las variables a su condición de reposo y apaga la lógica pendiente.

## 4. Evolución de las variables al usar la Interfaz

Cuando otro módulo llama a la función `put_event_task_actuator(event, identifier)` provista en `task_actuator_interface.c`, ocurre lo siguiente:

*   **`identifier`:** Es el argumento que entra a la función (ej. `ID_LED_A`). Se utiliza directamente como índice del arreglo `task_actuator_dta_list` para acceder a la memoria de ese actuador en particular.
*   **`task_actuator_dta_list[identifier].event`:** Se sobrescribe con el valor del parámetro `event` que llegó a la función (por ejemplo, `EV_LED_ACTIVE` o `EV_LED_IDLE`).
*   **`task_actuator_dta_list[identifier].flag`:** Se fuerza al valor `true` (booleano). Esto es crucial porque es la señal que necesita la función `task_actuator_statechart` en su próximo ciclo de reloj para enterarse de que el evento almacenado es nuevo y debe ser ejecutado.
