# Análisis de Código Fuente: Módulo Sensor y Cola de Eventos del Sistema

Este documento analiza el funcionamiento y la interacción entre los archivos `task_sensor_attribute.h`, `task_system_attribute.h`, `task_sensor.c` y `task_system_interface.c`, implementados bajo una arquitectura orientada a eventos con tareas no bloqueantes (*Update by Time Code* evaluado cada 1 milisegundo).

---

## 1. Funcionamiento General del Código

Los módulos analizados estructuran la capa de adquisición de entradas físicas (sensores) y el mecanismo asíncrono de comunicación hacia la lógica central del sistema:

*   **`task_sensor_attribute.h`**: Define las constantes, estructuras de datos y enumeraciones del módulo Sensor:
    *   Eventos físicos de entrada: `task_sensor_ev_t` (`EV_BTN_UP`, `EV_BTN_DOWN`).
    *   Estados de la FSM: `task_sensor_st_t` (`ST_BTN_IDLE`, `ST_BTN_ACTIVE`).
    *   Identificador del periférico: `task_sensor_id_t` (`ID_BTN_A`).
    *   Estructura de configuración estática (`task_sensor_cfg_t`): almacena puerto GPIO, pin, lógica de activación (`pressed`), retardo máximo y señales a emitir hacia System.
    *   Estructura dinámica de datos (`task_sensor_dta_t`): mantiene el tiempo transcurrido (`tick`), estado actual y último evento detectado.
*   **`task_system_attribute.h`**: Define los tipos de datos requeridos por la lógica central (System):
    *   Eventos consumidos: `task_system_ev_t` (`EV_SYS_IDLE`, `EV_SYS_ACTIVE`).
    *   Estados: `task_system_st_t` (`ST_SYS_IDLE`, `ST_SYS_ACTIVE`).
    *   Estructura de estado interna: `task_system_dta_t`.
*   **`task_sensor.c`**: Contiene la lógica ejecutiva del escrutinio de entradas:
    *   `task_sensor_init()`: Inicializa las variables de estado de cada sensor mapeado en la lista estática `task_sensor_cfg_list`.
    *   `task_sensor_update()`: Función invocada periódicamente desde el despachador de tareas (`app_update()`), la cual recorre todos los sensores disponibles y ejecuta su máquina de estados finitos.
    *   `task_sensor_statechart()`: Ejecuta la evaluación del hardware y las transiciones de estado del sensor seleccionado por índice.
*   **`task_system_interface.c`**: Implementa una cola circular FIFO (`event_task_system_queue_t`) de longitud fija (`QUEUE_LENGTH = 16`) que desacopla la generación de eventos de los sensores respecto al procesamiento de la tarea System.

---

## 2. Evolución de Variables del Módulo Sensor

A continuación se detalla cómo evolucionan las variables desde la inicialización (`task_sensor_init()`) y a lo largo de sucesivas invocaciones de `task_sensor_update()`:

### `index`
*   **En `task_sensor_init()`**: Es una variable local utilizada como índice de iteración. Comienza en `0` y se incrementa de a una unidad hasta llegar a `SENSOR_DTA_QTY` (en esta configuración, `1`), permitiendo configurar la estructura de cada sensor.
*   **En `task_sensor_update()`**: Se reinicia a `0` en cada invocación periódica y recorre el arreglo de sensores invocando `task_sensor_statechart(index)` para cada elemento configurado.

### `task_sensor_dta_list[index].tick` (Unidad de medida: milisegundos / ciclos de llamada)
*   **En `task_sensor_init()`**: No se le asigna un valor de forma explícita (mantiene el valor cero garantizado por la limpieza de la sección `.bss` en el arranque).
*   **En `task_sensor_update()` / `task_sensor_statechart()`**:
    *   En la implementación simplificada provista en `task_sensor.c`, el conteo temporal activo no está habilitado dentro de los estados `ST_BTN_IDLE` y `ST_BTN_ACTIVE`.
    *   En caso de alcanzarse el bloque `default` del `switch`, se fuerza a `DEL_BTN_MIN` (`0ul`).
    *   *Nota de diseño*: En el modelo completo con filtro antirrebote temporizado, esta variable adopta el valor `DEL_BTN_MAX` (`50ul`, equivalente a 50 ms) ante una transición y decrementa una unidad por cada ciclo de 1 ms hasta alcanzar `0`.

### `task_sensor_dta_list[index].state`
*   **En `task_sensor_init()`**: Se inicializa explícitamente en `ST_BTN_IDLE` (estado de reposo / inactivo).
*   **En `task_sensor_update()`**:
    *   Permanecerá en `ST_BTN_IDLE` mientras el botón no esté presionado.
    *   Al detectarse que el pulsador pasa al estado activo (`EV_BTN_DOWN`), transiciona a `ST_BTN_ACTIVE`.
    *   Permanecerá en `ST_BTN_ACTIVE` mientras el pulsador se mantenga accionado.
    *   Al liberarse el pulsador (`EV_BTN_UP`), transiciona nuevamente a `ST_BTN_IDLE`.
    *   Si por corrupción de memoria adopta un valor anómalo, el bloque `default` lo reasigna de manera segura a `ST_BTN_IDLE`.

### `task_sensor_dta_list[index].event`
*   **En `task_sensor_init()`**: Se inicializa explícitamente en `EV_BTN_UP`.
*   **En `task_sensor_update()`**: Al ingresar a `task_sensor_statechart()`, lee el nivel lógico del pin mediante `HAL_GPIO_ReadPin()`.
    *   Si la lectura coincide con `p_task_sensor_cfg->pressed`, se asigna inmediatamente `EV_BTN_DOWN`.
    *   Si la lectura no coincide, se asigna `EV_BTN_UP`.

---

## 3. Comportamiento de la Función `task_sensor_statechart(uint32_t index)`

La función `void task_sensor_statechart(uint32_t index)` ejecuta la lógica de lectura y la máquina de estados finitos asociada al sensor indexado:

1.  **Enlace de Punteros**: Obtiene referencias directas a la configuración física del sensor (`p_task_sensor_cfg`) y a su registro de variables dinámicas (`p_task_sensor_dta`).
2.  **Lectura y Mapeo de Hardware**:
    *   Consulta el pin GPIO configurado mediante `HAL_GPIO_ReadPin(p_task_sensor_cfg->gpio_port, p_task_sensor_cfg->pin)`.
    *   Si el nivel eléctrico equivale a la polaridad activa (`p_task_sensor_cfg->pressed`), establece `p_task_sensor_dta->event = EV_BTN_DOWN`. En caso contrario, establece `p_task_sensor_dta->event = EV_BTN_UP`.
3.  **Evaluación de Transiciones (Statechart)**:
    *   **Caso `ST_BTN_IDLE`**: Evalúa si `p_task_sensor_dta->event == EV_BTN_DOWN`. Si es verdadero, coloca la señal activa configurada en la cola del sistema mediante `put_event_task_system(p_task_sensor_cfg->signal_down)` (en este caso `EV_SYS_ACTIVE`) y cambia el estado a `ST_BTN_ACTIVE`.
    *   **Caso `ST_BTN_ACTIVE`**: Evalúa si `p_task_sensor_dta->event == EV_BTN_UP`. Si es verdadero, envía la señal inactiva configurada mediante `put_event_task_system(p_task_sensor_cfg->signal_up)` (en este caso `EV_SYS_IDLE`) y retorna al estado `ST_BTN_IDLE`.
    *   **Caso `default`**: Restablece los parámetros dinámicos a un estado conocido seguro (`tick = DEL_BTN_MIN`, `state = ST_BTN_IDLE`, `event = EV_BTN_UP`).

---

## 4. Evolución de las Variables de la Cola Circular (`event_task_system_queue`)

La cola de eventos permite almacenar las señales producidas por los sensores hasta que la tarea central las consuma. Sus variables evolucionan del siguiente modo:

### Inicialización (`init_event_task_system()`)
*   `event_task_system_queue.head = 0`: Apunta al índice donde se insertará el próximo evento.
*   `event_task_system_queue.tail = 0`: Apunta al índice de donde se extraerá el próximo evento.
*   `event_task_system_queue.count = 0`: Indica que la cola se encuentra vacía.
*   `event_task_system_queue.queue[i] = EMPTY` (`255ul`): Todas las posiciones del arreglo (de `0` a `15`) se inicializan con la constante `EMPTY`.

### Durante la Ejecución (`task_sensor_update()`)
Cada vez que el Statechart del sensor valida una transición y llama a `put_event_task_system(event)`:

*   **`event_task_system_queue.count`**:
    *   Se incrementa en una unidad (`count++`) por cada evento ingresado, reflejando la cantidad de eventos acumulados sin procesar.
*   **`event_task_system_queue.queue[head]`**:
    *   La posición apuntada por `head` sobrescribe su valor anterior (que era `EMPTY`) y almacena el nuevo evento (por ejemplo, `EV_SYS_ACTIVE` o `EV_SYS_IDLE`).
*   **`event_task_system_queue.head`**:
    *   Se incrementa en `1` post-inserción (`head++`).
    *   Si alcanza el límite máximo del buffer circular (`QUEUE_LENGTH = 16`), se reinicia a `0` para continuar insertando desde el inicio.
*   **`event_task_system_queue.tail`**:
    *   No sufre ninguna modificación durante `task_sensor_update()`.
    *   Solo se modificará cuando la tarea del sistema invoque `get_event_task_system()`, momento en el cual `tail` avanzará hacia la siguiente posición y la celda leída volverá a quedar marcada con `EMPTY`.