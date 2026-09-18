# Descripción de la solución de COMA Electronics

De acuerdo con la referencia "3.1.- TA134 - TdSE - 1er Proyecto", la solución de COMA Electronics se estructura en un ecosistema integral conocido como **Intelligent Parking Management System**. Este consta de los siguientes elementos principales:

*   **Parking System Server:** Servidor central encargado de la gestión general del sistema.
*   **Entry Machine & Exit Machine:** Terminales ubicadas en los accesos de entrada y salida, con diversas opciones de configuración disponibles según la necesidad del cliente.
*   **Toll Computer / Automatic Pay Station:** Estaciones dedicadas exclusivamente al cobro y validación de los pagos.

## Automated Parking System (Flujo del Sistema)
Dentro de este ecosistema funciona el flujo automatizado, cuyo comportamiento estándar es el siguiente:

1. El vehículo arriba a la terminal de entrada (*Entry Machine*).
2. El conductor presiona un botón, desencadenando la emisión de un ticket o tarjeta que contiene un número de serie, la fecha y la hora. Simultáneamente, se envía una señal para abrir la barrera y permitir el ingreso del vehículo.
3. Antes de retirarse del establecimiento, el cliente debe llevar su ticket a un punto de pago central o estación automática para abonar y validarlo.
4. Finalmente, al llegar a la terminal de salida (*Exit Machine*), el sistema lee el ticket validado y envía la orden a la barrera para que se abra, permitiendo la salida del vehículo.

## Parking Ticket Dispenser Machine (Entry)
El foco de este proyecto se centra en la terminal de entrada. El hardware estipulado para esta máquina incluye:

*   Pantalla LCD de 7 pulgadas.
*   Lector de tarjetas y botón de ayuda.
*   Ranura dispensadora y botón para solicitar el ticket.
*   Sistema de avisos por voz (*Voice prompt*).
*   Intercomunicador (opcional).
*   A nivel de infraestructura de carril, se integra con una cámara motorizada con luz automática, una barrera de alta velocidad (ambas activables mediante un disparador por radar) y un display LED para indicar la cantidad de cupos vacantes.

## Arquitectura de Implementación
Para llevar a cabo el control de esta terminal mediante software embebido, se utiliza una arquitectura modular diseñada para evitar bloqueos y asegurar un comportamiento concurrente. Se divide en tres etapas funcionales:

1.  **Escrutar (Scrutinize):** Módulo encargado de monitorear los sensores y entradas digitales (cámara, botón de ticket, bobina sensora).
2.  **Procesar (Process):** Módulo central que recibe la información del entorno, ejecuta la máquina de estados lógicos y toma decisiones.
3.  **Actuar (Act):** Módulo responsable de traducir las decisiones lógicas en modificaciones sobre las salidas digitales o actuadores (pantalla, impresora, motor de la barrera y notificaciones al servidor).

**Sincronización y Ejecución:** 
Los tres módulos son independientes y se comunican de forma exclusiva a través del intercambio de mensajes. Todo el sistema se rige bajo un esquema de ejecución cíclica de tareas no bloqueantes (*Update by Time Code*), donde cada máquina de estados es evaluada estrictamente cada 1 milisegundo, garantizando que el uso del procesador sea equitativo y evitando cuelgues del sistema.