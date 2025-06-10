# P4

# PRACTICA 4 :  SISTEMAS OPERATIVOS EN TIEMPO REAL  

El objetivo de la practica es comprender el funcionamiento de un sistema operativo en tiempo 
Real .

Para lo cual realizaremos una practica  donde  generaremos varias tareas  y veremos como 
se ejecutan dividiendo el tiempo de uso de la cpu.



## Introducción teórica  



## Multitarea en ESP32 con Arduino y FreeRTOS

A estas alturas, no es ningún secreto que el ESP32 es mi chip de referencia para fabricar dispositivos IoT. 
Son pequeños, potentes, tienen un montón de funciones integradas y son relativamente fáciles de programar.

Sin embargo, cuando se usa junto con Arduino, todo su código se ejecuta en un solo núcleo. Eso parece un poco 
desperdicio, así que cambiemos eso usando FreeRTOS para programar tareas en ambos núcleos.

### ¿Por qué?
Hay varios casos de uso para querer realizar múltiples tareas en un microcontrolador. Por ejemplo: 
puede tener un microcontrolador que lee un sensor de temperatura, lo muestra en una pantalla LCD y lo envía a la nube.

Puede hacer los tres sincrónicamente, uno tras otro. Pero, ¿qué sucede si está utilizando una pantalla de tinta electrónica que 
tarda unos segundos en actualizarse?
    **Responder en el informe**

Afortunadamente, la implementación de Arduino para ESP32 incluye la posibilidad de programar tareas con FreeRTOS. Estos pueden
 ejecutarse en un solo núcleo, muchos núcleos e incluso puede definir cuál es más importante y debe recibir un trato preferencial.

### Creando tareas
Para programar una tarea, debe hacer dos cosas: crear una función que contenga el código que desea ejecutar y luego crear una
 tarea que llame a esta función.

Digamos que quiero hacer parpadear un LED encendido y apagado continuamente.

Primero, definiré el pin al que está conectado el LED y estableceré su modo OUTPUT. Cosas muy estándar de Arduino:
```
const int led1 = 2; // Pin of the LED

void setup(){
  pinMode(led1, OUTPUT);
}
```
A continuación, crearé una función que se convertirá en la base de la tarea. Utilizo digitalWrite()para encender y apagar el LED y
 usar **vTaskDelay** (en lugar de **delay()** ) para pausar la tarea 500ms entre estados cambiantes:
```
void toggleLED(void * parameter){
  for(;;){ // infinite loop

    // Turn the LED on
    digitalWrite(led1, HIGH);

    // Pause the task for 500ms
    vTaskDelay(500 / portTICK_PERIOD_MS);

    // Turn the LED off
    digitalWrite(led1, LOW);

    // Pause the task again for 500ms
    vTaskDelay(500 / portTICK_PERIOD_MS);
  }
}
```
¡Esa es tu primera tarea! Un par de cosas a tener en cuenta:

Sí, creamos un for(;;)bucle infinito , y eso puede parecer un poco extraño. ¿Cómo podemos realizar múltiples tareas si escribimos una 
tarea que continúa para siempre? El truco es **vTaskDelay** que le dice al programador que esta tarea no debe ejecutarse durante un 
período determinado. El planificador pausará el ciclo for y ejecutará otras tareas (si las hay).

Por último, pero no menos importante, tenemos que informarle al planificador sobre nuestra tarea. Podemos hacer esto en la setup()
función:
```
void setup() {
  xTaskCreate(
    toggleLED,    // Function that should be called
    "Toggle LED",   // Name of the task (for debugging)
    1000,            // Stack size (bytes)
    NULL,            // Parameter to pass
    1,               // Task priority
    NULL             // Task handle
  );
}
```

¡Eso es! ¿Quiere hacer parpadear otro LED en un intervalo diferente? Simplemente cree otra tarea y siéntese mientras el programador se encarga de ejecutar ambas.

Crear una tarea única
También puede crear tareas que solo se ejecuten una vez. Por ejemplo, mi monitor de energía crea una tarea para cargar datos en la nube cuando tiene suficientes lecturas.

Las tareas únicas no necesitan un ciclo for interminable, sino que se ve así:
```
void uploadToAWS(void * parameter){
    // Implement your custom logic here

    // When you're done, call vTaskDelete. Don't forget this!
    vTaskDelete(NULL);
}
```

Esto parece una función normal de C ++ excepto por vTaskDelete(). Después de llamarlo, FreeRTOS sabe que la tarea ha finalizado y no debe reprogramarse. (Nota: no olvide llamar a esta función o hará que el perro guardián reinicie el ESP32).
```
xTaskCreate(
    uploadToAWS,    // Function that should be called
    "Upload to AWS",  // Name of the task (for debugging)
    1000,            // Stack size (bytes)
    NULL,            // Parameter to pass
    1,               // Task priority
    NULL             // Task handle
);
```
### Elija en qué núcleo se ejecutará
Cuando lo usa **xTaskCreate()**, el programador es libre de elegir en qué núcleo ejecuta su tarea. En mi opinión, esta es la solución
 más flexible (nunca se sabe cuándo puede aparecer un chip IoT de cuatro núcleos, ¿verdad?)

Sin embargo, es posible anclar una tarea a un núcleo específico con **xTaskCreatePinnedToCore**. Es como **xTaskCreatey** toma un 
parámetro adicional, el núcleo en el que desea ejecutar la tarea:

```
xTaskCreatePinnedToCore(
    uploadToAWS,      // Function that should be called
    "Upload to AWS",    // Name of the task (for debugging)
    1000,               // Stack size (bytes)
    NULL,               // Parameter to pass
    1,                  // Task priority
    NULL,               // Task handle
    0,          // Core you want to run the task on (0 or 1)
);
```

### Compruebe en qué núcleo se está ejecutando
La mayoría de las placas ESP32 tienen procesadores de doble núcleo, entonces, ¿cómo sabe en qué núcleo se está ejecutando su tarea?

Simplemente llame xPortGetCoreID()desde dentro de su tarea:
```
void exampleTask(void * parameter){
  Serial.print("Task is running on: ");
  Serial.println(xPortGetCoreID());
  vTaskDelay(100 / portTICK_PERIOD_MS);
}
```
Cuando tenga suficientes tareas, el programador comenzará a enviarlas a ambos núcleos.

### Detener tareas
Ahora, ¿qué pasa si agregó una tarea al programador, pero desea detenerla? Dos opciones: borra la tarea desde dentro o usa un 
identificador de tareas. Terminar una tarea desde adentro ya se discutió antes (uso vTaskDelete).

Para detener una tarea desde otro lugar (como otra tarea o su bucle principal), tenemos que almacenar un identificador de tarea:
```
// This TaskHandle will allow 
TaskHandle_t task1Handle = NULL;

void task1(void * parameter){
   // your task logic
}

xTaskCreate(
    task1,
    "Task 1",
    1000,
    NULL,
    1,
    task1Handle            // Task handle
);
```
Todo lo que tuvimos que hacer fue definir el identificador y pasarlo como último parámetro de xTaskCreate. Ahora podemos matarlo con 
vTaskDelete:
```
void anotherTask(void * parameter){
  // Kill task1 if it's running
  if(task1Handle != NULL) {
    vTaskDelete(task1Handle);
  }
}
```
### Prioridad de la tarea
A la hora de crear tareas, tenemos que darle prioridad. Es el quinto parámetro de xTaskCreate. Las prioridades son importantes cuando
 dos o más tareas compiten por el tiempo de la CPU. Cuando eso suceda, el programador ejecutará primero la tarea de mayor prioridad.
  ¡Tiene sentido!

En FreeRTOS, un número de prioridad más alto significa que una tarea es más importante . Encontré esto un poco contra-intuitivo 
porque para mí una "prioridad 1" suena más importante que una "prioridad 2", pero ese soy solo yo.

Cuando dos tareas comparten la misma prioridad, FreeRTOS compartirá el tiempo de procesamiento disponible entre ellas.

Cada tarea puede tener una prioridad entre 0 y 24. El límite superior está definido por configMAX_PRIORITIESen el archivo 
FreeRTOSConfig.h .

Utilizo esto para diferenciar las tareas primarias de las secundarias. Tome el medidor de energía de mi casa: la tarea de mayor 
prioridad es medir la electricidad (prioridad 3). Actualizar la pantalla o sincronizar la hora con un servidor NTP no es tan crítico 
para su funcionalidad principal (prioridad 2).



## Ejercicio Practico 1 

Programar el siguiente codigo 


```
void setup()
{
Serial.begin(112500);
/* we create a new task here */
xTaskCreate(
anotherTask, /* Task function. */
"another Task", /* name of task. */
10000, /* Stack size of task */
NULL, /* parameter of the task */
1, /* priority of the task */
NULL); /* Task handle to keep track of created task */
}
 
/* the forever loop() function is invoked by Arduino ESP32 loopTask */
void loop()
{
Serial.println("this is ESP32 Task");
delay(1000);
}
 
/* this function will be invoked when additionalTask was created */
void anotherTask( void * parameter )
{
/* loop forever */
for(;;)
{
Serial.println("this is another Task");
delay(1000);
}
/* delete a task when finish,
this will never happen because this is infinity loop */
vTaskDelete( NULL );
}

```
 

## Codigp

/*
* ESP32 RTOS - Reloj Digital con Pulsadores y LEDs
*
* Este proyecto implementa un reloj digital utilizando FreeRTOS en un ESP32.
* Características:
* - Muestra horas, minutos y segundos
* - Permite ajustar el tiempo mediante pulsadores
* - Utiliza LEDs para indicar estados del reloj
* - Implementa múltiples tareas concurrentes con RTOS
*/
#include <Arduino.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "freertos/semphr.h"
// Definición de pines
#define LED_SEGUNDOS 4 // LED que parpadea cada segundo
#define LED_MODO 5 // LED que indica modo de ajuste
#define BTN_MODO 6 // Botón para cambiar modo (normal/ajuste hora/ajuste minutos)
#define BTN_INCREMENTO 17 // Botón para incrementar valor en modo ajuste
// Variables para el reloj
volatile int horas = 0;
volatile int minutos = 0;
volatile int segundos = 0;
volatile int modo = 0; // 0: modo normal, 1: ajuste horas, 2: ajuste minutos
// Variables FreeRTOS
QueueHandle_t botonQueue;
SemaphoreHandle_t relojMutex;
// Estructura para los eventos de botones
typedef struct {
uint8_t boton;
uint32_t tiempo;
} EventoBoton;
// Prototipos de tareas
void TareaReloj(void *pvParameters);
void TareaLecturaBotones(void *pvParameters);
void TareaActualizacionDisplay(void *pvParameters);
void TareaControlLEDs(void *pvParameters);
// Función para manejar interrupciones de botones
void IRAM_ATTR ISR_Boton(void *arg) {
uint8_t numeroPulsador = (uint32_t)arg;
// Crear un evento para el botón
EventoBoton evento;
evento.boton = numeroPulsador;
evento.tiempo = xTaskGetTickCountFromISR();
// Enviar evento a la cola
xQueueSendFromISR(botonQueue, &evento, NULL);
}
void setup() {
// Inicializar puerto serie
Serial.begin(115200);
Serial.println("Inicializando Reloj Digital con RTOS");
// Configurar pines
pinMode(LED_SEGUNDOS, OUTPUT);
pinMode(LED_MODO, OUTPUT);
pinMode(BTN_MODO, INPUT_PULLUP);
pinMode(BTN_INCREMENTO, INPUT_PULLUP);
// Crear recursos RTOS
botonQueue = xQueueCreate(10, sizeof(EventoBoton));
relojMutex = xSemaphoreCreateMutex();
// Configurar interrupciones para los botones
attachInterruptArg(BTN_MODO, ISR_Boton, (void*)BTN_MODO, FALLING);
attachInterruptArg(BTN_INCREMENTO, ISR_Boton, (void*)BTN_INCREMENTO,
FALLING);
// Crear tareas
xTaskCreate(
TareaReloj, // Función que implementa la tarea
"RelojTask", // Nombre de la tarea
2048, // Tamaño de la pila en palabras
NULL, // Parámetros de entrada
1, // Prioridad
NULL // Handle de la tarea
);
xTaskCreate(
TareaLecturaBotones,
"BotonesTask",
2048,
NULL,
2, // Mayor prioridad para respuesta rápida
NULL
);
xTaskCreate(
TareaActualizacionDisplay,
"DisplayTask",
2048,
NULL,
1,
NULL
);
xTaskCreate(
TareaControlLEDs,
"LEDsTask",
1024,
NULL,
1,
NULL
);
// El planificador RTOS se inicia automáticamente
}
void loop() {
// loop() queda vacío en aplicaciones RTOS
// Todo el trabajo se hace en las tareas
vTaskDelay(portMAX_DELAY);
}
// Tarea que actualiza el tiempo del reloj
void TareaReloj(void *pvParameters) {
TickType_t xLastWakeTime;
const TickType_t xPeriod = pdMS_TO_TICKS(1000); // Periodo de 1 segundo
// Inicializar xLastWakeTime con el tiempo actual
xLastWakeTime = xTaskGetTickCount();
for (;;) {
// Esperar hasta el siguiente periodo
vTaskDelayUntil(&xLastWakeTime, xPeriod);
// Obtener semáforo para acceder a las variables del reloj
if (xSemaphoreTake(relojMutex, portMAX_DELAY) == pdTRUE) {
// Actualizar solo en modo normal
if (modo == 0) {
segundos++;
if (segundos >= 60) {
segundos = 0;
minutos++;
if (minutos >= 60) {
minutos = 0;
horas++;
if (horas >= 24) {
horas = 0;
}
}
}
}
// Liberar el semáforo
xSemaphoreGive(relojMutex);
}
}
}
// Tarea que gestiona los botones y sus acciones
void TareaLecturaBotones(void *pvParameters) {
EventoBoton evento;
uint32_t ultimoTiempoBoton = 0;
const uint32_t debounceTime = pdMS_TO_TICKS(300); // Tiempo anti-rebote
for (;;) {
// Esperar a que llegue un evento desde la interrupción
if (xQueueReceive(botonQueue, &evento, portMAX_DELAY) == pdPASS) {
// Verificar si ha pasado suficiente tiempo desde la última pulsación (anti-rebote)
if ((evento.tiempo - ultimoTiempoBoton) >= debounceTime) {
// Obtener semáforo para modificar variables del reloj
if (xSemaphoreTake(relojMutex, portMAX_DELAY) == pdTRUE) {
// Procesar el evento según el botón
if (evento.boton == BTN_MODO) {
// Cambiar modo
modo = (modo + 1) % 3;
Serial.printf("Cambio de modo: %d\n", modo);
}
else if (evento.boton == BTN_INCREMENTO) {
// Incrementar valor según el modo actual
if (modo == 1) { // Ajuste de horas
horas = (horas + 1) % 24;
Serial.printf("Horas ajustadas a: %d\n", horas);
}
else if (modo == 2) { // Ajuste de minutos
minutos = (minutos + 1) % 60;
segundos = 0; // Reiniciar segundos al cambiar minutos
Serial.printf("Minutos ajustados a: %d\n", minutos);
}
}
// Liberar el semáforo
xSemaphoreGive(relojMutex);
}
// Actualizar tiempo de la última pulsación
ultimoTiempoBoton = evento.tiempo;
}
}
}
}
// Tarea que actualiza la información en el display (puerto serie en este caso)
void TareaActualizacionDisplay(void *pvParameters) {
int horasAnterior = -1;
int minutosAnterior = -1;
int segundosAnterior = -1;
int modoAnterior = -1;
for (;;) {
// Obtener semáforo para leer variables del reloj
if (xSemaphoreTake(relojMutex, portMAX_DELAY) == pdTRUE) {
// Verificar si hay cambios que mostrar
bool cambios = (horas != horasAnterior) ||
(minutos != minutosAnterior) ||
(segundos != segundosAnterior) ||
(modo != modoAnterior);
if (cambios) {
// Imprimir información del reloj
Serial.printf("%02d:%02d:%02d", horas, minutos, segundos);
// Mostrar el modo actual
if (modo == 0) {
Serial.println(" [Modo Normal]");
} else if (modo == 1) {
Serial.println(" [Ajuste Horas]");
} else if (modo == 2) {
Serial.println(" [Ajuste Minutos]");
}
// Actualizar valores anteriores
horasAnterior = horas;
minutosAnterior = minutos;
segundosAnterior = segundos;
modoAnterior = modo;
}
// Liberar el semáforo
xSemaphoreGive(relojMutex);
}
// Esperar antes de la siguiente actualización
vTaskDelay(pdMS_TO_TICKS(100));
}
}
// Tarea que controla los LEDs según el estado del reloj
void TareaControlLEDs(void *pvParameters) {
bool estadoLedSegundos = false;
for (;;) {
// Obtener semáforo para leer variables del reloj
if (xSemaphoreTake(relojMutex, portMAX_DELAY) == pdTRUE) {
// LED de segundos parpadea cada segundo
if (segundos != estadoLedSegundos) {
estadoLedSegundos = !estadoLedSegundos;
digitalWrite(LED_SEGUNDOS, estadoLedSegundos);
}
// LED de modo se enciende solo en modo ajuste
digitalWrite(LED_MODO, modo > 0);
// Liberar el semáforo
xSemaphoreGive(relojMutex);
}
// Esperar antes de la siguiente actualización
vTaskDelay(pdMS_TO_TICKS(50));
}
}






