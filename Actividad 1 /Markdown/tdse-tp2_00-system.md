# Análisis de Código Fuente: Módulo System y Actuador

Este documento detalla el funcionamiento y la interacción de los archivos que componen el núcleo lógico (System) del sistema embebido y sus interfaces de comunicación, implementados bajo una arquitectura orientada a eventos de ejecución no bloqueante (*Update by Time Code*).

## 1. Análisis del funcionamiento del código fuente

El código adjunto implementa el núcleo lógico (System) y sus interfaces de comunicación:

* **`task_system_attribute.h` y `task_system_interface.h`/`.c`:** Definen los datos del sistema y una cola circular (FIFO) que actúa como buzón de mensajes. Esto permite que otros módulos (como el Sensor) inserten eventos (`EV_SYS_IDLE`, `EV_SYS_ACTIVE`) de forma asíncrona para que el Sistema los procese posteriormente.
* **`task_system.c`:** Contiene la inicialización de la tarea central y la máquina de estados principal. En cada ciclo de `task_system_update()`, el sistema consulta si hay eventos en su cola y, dependiendo de su estado actual, ejecuta la lógica de negocio y envía comandos de salida.
* **`task_actuator_interface.c`:** Expone la función `put_event_task_actuator()` que el módulo System utiliza para enviar señales directas a los actuadores (como encender o apagar un LED).

## 2. Evolución de las variables del Sistema (System)

Al ejecutarse `task_system_init()` y durante el bucle de `task_system_update()`, las variables de `task_system_dta_list` evolucionan así:

* **`index`:** En `task_system_init()`, actúa como iterador de un bucle `for` que va de 0 hasta `SYSTEM_DTA_QTY - 1` (donde `SYSTEM_DTA_QTY` es 1 por el modo `NORMAL`). Por lo tanto, `index` toma el valor 0 para inicializar la única instancia del sistema.
* **`task_system_dta_list[index].tick` (Unidad: Milisegundos o ciclos):** No se inicializa explícitamente en el inicio. Si la máquina de estados cae en el caso `default`, se le asigna `DEL_SYS_MIN` (0). En este código específico, la variable no presenta incrementos activos, sirviendo como un resguardo de la estructura.
* **`task_system_dta_list[index].state`:** Se inicializa en `ST_SYS_IDLE`. En las ejecuciones de `update()`, si se procesa el evento `EV_SYS_ACTIVE`, cambia a `ST_SYS_ACTIVE`. Si estando activo recibe `EV_SYS_IDLE`, regresa a `ST_SYS_IDLE`.
* **`task_system_dta_list[index].event`:** Arranca inicializado en `EV_SYS_IDLE`. Durante `task_system_update()`, adopta el valor de los eventos que son extraídos de la cola mediante `get_event_task_system()`.
* **`task_system_dta_list[index].flag`:** Comienza en `false`. Cambia a `true` al extraer un nuevo evento de la cola, indicando que hay información fresca por procesar. Vuelve a `false` inmediatamente después de consumirse el evento en la transición de estado correspondiente.

## 3. Comportamiento de la función Statechart

*(Nota: En el archivo adjunto `task_system.c`, la función correspondiente se denomina `task_system_normal_statechart(void)`, la cual aborda el índice `NORMAL` implícitamente en lugar de recibirlo por parámetro).*

1. **Lectura de eventos:** La función consulta `any_event_task_system()`. Si devuelve `true`, extrae el evento de la cola con `get_event_task_system()`, lo guarda en la variable `event` del estado y levanta el `flag` a `true`.
2. **Evaluación de estados (switch-case):**
   * Si está en **`ST_SYS_IDLE`** y detecta `flag == true` junto con el evento `EV_SYS_ACTIVE`, baja el flag, envía una señal al actuador (`EV_LED_ACTIVE`, `ID_LED_A`) y cambia al estado `ST_SYS_ACTIVE`.
   * Si está en **`ST_SYS_ACTIVE`** y recibe `EV_SYS_IDLE` con el flag en alto, baja el flag, envía la señal contraria al actuador (`EV_LED_IDLE`) y retorna a `ST_SYS_IDLE`.
   * Si cae en **`default`**, resetea las variables del sistema por seguridad: estado a IDLE, evento a IDLE, flag a false y tick a 0.

## 4. Evolución de la cola de eventos del Sistema

Al invocar `init_event_task_system()` y recibir/extraer eventos:

* **`i`:** Es un iterador local en la inicialización que recorre de 0 a 15 (`QUEUE_LENGTH - 1`) para llenar la cola de valores vacíos.
* **`event_task_system_queue.head`:** Inicia en 0. Al invocar `put_event_task_system()` (por un módulo externo), incrementa en 1, apuntando a la próxima celda libre. Si llega a 16 (`QUEUE_LENGTH`), se reinicia a 0 (comportamiento circular).
* **`event_task_system_queue.tail`:** Inicia en 0. Al invocar `get_event_task_system()` (desde el statechart), incrementa en 1, avanzando hacia el siguiente mensaje pendiente. También vuelve a 0 al alcanzar 16.
* **`event_task_system_queue.count`:** Inicia en 0. Aumenta en 1 con cada `put` y disminuye en 1 con cada `get`, reflejando la cantidad exacta de eventos sin leer.
* **`event_task_system_queue.queue[i]`:** Todo el arreglo se inicializa con el valor 255 (`EMPTY`). Al hacer un `put`, la celda apuntada por `head` se sobrescribe con el evento (ej. `EV_SYS_ACTIVE`). Al hacer un `get`, el evento es extraído y la celda apuntada por `tail` vuelve a marcarse como 255 (`EMPTY`).

## 5. Evolución de las variables del Actuador

Estas variables se ven afectadas indirectamente cuando `task_system_normal_statechart()` llama a `put_event_task_actuator(event, identifier)`:

* **`identifier`:** Es el argumento pasado a la función (por ejemplo, `ID_LED_A` desde el sistema), el cual se usa como índice para ubicar qué actuador específico se debe modificar en el arreglo `task_actuator_dta_list`.
* **`task_actuator_dta_list[identifier].event`:** Se sobrescribe con el evento que le envía el Sistema (por ejemplo, `EV_LED_ACTIVE` o `EV_LED_IDLE`) indicando la nueva orden física a ejecutar.
* **`task_actuator_dta_list[identifier].flag`:** Cada vez que la función es invocada, este valor se fuerza a `true`, notificándole a la máquina de estados del Actuator que hay una nueva orden pendiente de atención.
