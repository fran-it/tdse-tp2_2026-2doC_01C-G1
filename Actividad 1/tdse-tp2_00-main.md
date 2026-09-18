# Análisis de Código Fuente STM32

A continuación se detalla el funcionamiento del código fuente correspondiente al arranque y la lógica principal del microcontrolador, así como la evolución de sus variables de tiempo y reloj.

## 1. Funcionamiento General del Código

El flujo de ejecución del programa avanza a través de los tres archivos de la siguiente manera:

*   **Arranque en ensamblador (`startup_stm32f103rbtx.s`):** Todo comienza en la rutina `Reset_Handler` inmediatamente después de encender o reiniciar el microcontrolador. Este código llama a la función `SystemInit` para establecer la configuración base del reloj. A continuación, prepara la memoria RAM copiando los valores iniciales de la sección `.data` desde la memoria Flash y llenando con ceros la sección `.bss` mediante el bucle `FillZerobss`. Finalmente, llama a la inicialización de la librería en C y salta a la función principal `main` mediante la instrucción `bl main`.
*   **Inicialización y Lógica Principal (`main.c`):** Al ingresar a la función `main()`, el programa ejecuta `HAL_Init()`, lo cual reinicia todos los periféricos e inicializa la interfaz Flash y el temporizador SysTick. Inmediatamente después, invoca a `SystemClock_Config()` para configurar la velocidad definitiva de los relojes del sistema. Luego inicializa los puertos GPIO y la comunicación serial USART2, para posteriormente ejecutar la inicialización de la aplicación a través de `app_init()`. Finalmente, el microcontrolador entra en el bucle infinito `while (1)`, donde ejecuta cíclicamente la rutina `app_update()`.
*   **Manejo de Interrupciones (`stm32f1xx_it.c`):** Este archivo aloja las rutinas de servicio de interrupción (ISR) a las que el procesador salta cuando ocurre un evento de hardware. Contiene manejadores como `SysTick_Handler`, que llama a `HAL_IncTick()` para actualizar el contador de tiempo base de las librerías HAL. También atiende interrupciones externas, como `EXTI15_10_IRQHandler`, la cual llama al manejador de GPIO para el pin del botón `B1_Pin`.

## 2. Evolución de la variable `SystemCoreClock`

*   **En el `Reset_Handler`:** Durante los primeros instantes de ejecución antes de llegar a C, el sistema funciona con el reloj interno por defecto (usualmente el HSI a 8 MHz para la familia F1).
*   **Durante `main()`:** Al ejecutarse la función `SystemClock_Config()`, el código activa el oscilador interno (HSI) y enciende el PLL. El PLL se configura utilizando el HSI dividido por 2 como fuente de entrada y aplicando un multiplicador de 16 (`RCC_PLL_MUL16`). Finalmente, se selecciona el reloj generado por el PLL como la fuente principal del sistema (`RCC_SYSCLKSOURCE_PLLCLK`). Esto modifica drásticamente la frecuencia de operación, momento en el cual la variable `SystemCoreClock` (utilizada por las librerías CMSIS para rastrear la velocidad) pasa a adoptar la frecuencia final calculada para el microcontrolador.

## 3. Evolución de la variable `SysTick`

*   **En el `Reset_Handler`:** El hardware del temporizador SysTick se encuentra desactivado y no genera interrupciones.
*   **Durante la inicialización en `main()`:** Al ejecutarse la instrucción `HAL_Init()`, el hardware del SysTick es configurado y encendido para generar una interrupción de forma periódica.
*   **En el bucle `while (1)`:** A partir de la inicialización, independientemente de lo que esté ejecutando el bucle infinito de `main.c`, el hardware interrumpe al procesador periódicamente para saltar a la función `SysTick_Handler()` en el archivo `stm32f1xx_it.c`. Dentro de este manejador, la llamada a `HAL_IncTick()` incrementa la variable interna que lleva la cuenta del tiempo transcurrido desde el encendido del sistema.
