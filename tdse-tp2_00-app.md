# Sistema Inteligente de Gestión de Estacionamiento

## Descripción de la solución de COMA Electronics
De acuerdo con la referencia de diseño del proyecto, la solución de COMA Electronics se estructura en un ecosistema integral conocido como **Intelligent Parking Management System**. Este consta de los siguientes elementos principales:

*   **Parking System Server:** Servidor central encargado de la gestión general del sistema y almacenamiento de datos.
*   **Entry Machine & Exit Machine:** Terminales ubicadas en los accesos de entrada y salida del predio, con diversas opciones de configuración disponibles según los requerimientos del lugar.
*   **Toll Computer / Automatic Pay Station:** Estaciones dedicadas exclusivamente al cobro y validación de los pagos por el tiempo de estadía.

## Automated Parking System (Flujo del Sistema)
Dentro de este ecosistema funciona el flujo automatizado, cuyo comportamiento estándar se divide en las siguientes etapas:

1.  **Ingreso:** El vehículo arriba a la terminal de entrada (*Entry Machine*).
2.  **Emisión:** El conductor presiona un botón, desencadenando la emisión de un ticket o tarjeta que contiene un número de serie único, la fecha y la hora de ingreso. Simultáneamente, el sistema envía una señal para abrir la barrera y permitir el paso del vehículo.
3.  **Validación y Pago:** Antes de retirarse del establecimiento, el cliente debe llevar su ticket a un punto de pago central o estación automática para abonar el servicio y validar su salida.
4.  **Salida:** Finalmente, al llegar a la terminal de salida (*Exit Machine*), el sistema lee el ticket validado y envía la orden a la barrera para que se abra, permitiendo la salida del predio.

## Parking Ticket Dispenser Machine (Entry)
El foco de la implementación técnica se centra en la terminal de entrada. A nivel de hardware, esta máquina está equipada de forma estándar con:

*   Pantalla LCD de 7 pulgadas.
*   Lector de tarjetas y botón de ayuda.
*   Ranura dispensadora de tickets y botón de solicitud.
*   Sistema de avisos por voz (*Voice prompt*).
*   Intercomunicador (opcional).

A nivel de infraestructura del carril de acceso, el sistema se integra con una cámara motorizada con luz automática, una barrera de alta velocidad (ambas activables mediante un disparador por radar o bobina sensora) y un display LED para indicar la cantidad de cupos vacantes en el estacionamiento.

### Arquitectura de Implementación (Software)
Para llevar a cabo el control de esta terminal mediante software embebido, se utiliza una arquitectura modular diseñada para evitar bloqueos y asegurar un comportamiento concurrente. Se divide en tres etapas funcionales:

1.  **Escrutar (Scrutinize) - Módulo Sensor:** Encargado de monitorear las entradas digitales del entorno (emulando la cámara, el botón de ticket y la bobina sensora mediante interruptores y pulsadores).
2.  **Procesar (Process) - Módulo System:** El núcleo lógico central que recibe los mensajes del módulo de escrutinio, ejecuta la máquina de estados de negocio y toma las decisiones.
3.  **Actuar (Act) - Módulo Actuator:** Responsable de traducir las decisiones lógicas en activaciones sobre las salidas digitales (emulando la pantalla, impresora, barrera y notificaciones al servidor mediante LEDs).

Todos los módulos operan bajo un esquema de ejecución cíclica de tareas no bloqueantes (*Update by Time Code*), evaluadas estrictamente cada 1 milisegundo, comunicándose exclusivamente mediante el paso de mensajes/señales.