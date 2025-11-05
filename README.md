# Contexto y objetivos
En este taller tomamos el rol de un equipo de ingenieros enfocados en el control y la automatización dentro de una empresa. La empresa tiene como objetivo desarrollar sistemas que supervisen y controlen entornos domésticos en tiempo real. Como parte de un producto, el equipo tiene la tarea de crear un sistema de supervisión y alerta de humedad para integrarse con el ecosistema IoT.

Diseñamos un prototipo que mide humedad y genera alertas remotas mediante IoT. Se implementó una arquitectura I2C maestro-esclavo  y un dashboard en ThingSpeak para ver la humedad actual, alertas y la historia de datos.
# Requisitos del sistema
* __Maestro:__ Recibe la medida de humedad, decide si mandar una alerta según un umbral propuesto (humedad > 20%) y pública a ThingSpeak el valor actual, la alerta y la gráfica.
* __Esclavo:__ Mide la humedad y envía el dato por I2C al maestro.
* __Entregables clave:__ Diagrama de actividad, documentación del código y resultados de pruebas.
# Arquitectura propuesta
## Hardware
* Dos Arduinos modelo MCO-446700374.
* Sensor de humedad de sustrato fc-28.
* Modulo wi-fi esp015.
* Protoboard.

<img width="1052" height="590" alt="image" src="https://github.com/user-attachments/assets/92bccc98-8665-4a18-b295-5bab04b794c5" />

## Software
* Script en C# para arduino encargado de la lectura de la humedad, filtrado, registro local y respuesta a read I2C.
* Script en C# para arduino que por medio de un loop solicita el porcentaje de humedad, evalúa si es necesario mandar una alerta basado en el umbral y envía la información a ThingSpeak para su actualización.
* Dashboard en ThingSpeak con panel de humedad actual, gráfica de humedad a través del tiempo e indicador de alerta.

<img width="1234" height="790" alt="image" src="https://github.com/user-attachments/assets/4e7b8740-6c0b-4b8d-bf3d-03588b80d94b" />

# Diagrama de actividad

<img width="497" height="897" alt="DiagramaActividad" src="https://github.com/user-attachments/assets/e1812dd6-e3ac-408d-ad99-24b9f24bd5fa" />

#Pruebas y validación
Para probar la comunicación I2C se realizó un print para visualizar la lectura de humedad en la consola; se revisaron iteradas lecturas teniendo el sensor seco y envuelto en un paño húmedo.

Para probar la conexión con ThingSpeak, primero fue necesario implementar el sensor de humedad. Se utilizó el panel de lectura y se revisó que se mostrará el mismo valor que en consola local.

Para probar la alerta se realizó un trigger por medio de código para poner en estado de alerta al sistema sin necesidad de un sensor. Luego se configuró la variable para guardar el umbral y por medio de una condición se activó el trigger. Primero se imprimió en consola un print de debug para saber si entraba en alerta y luego se conectó con el dashboard de ThingSpeak.
# Resultados
La arquitectura I2C se implementó y funcionó correctamente; el Arduino esclavo midió el porcentaje de humedad de forma estable y el Arduino maestro recibió el dato, evaluó la alerta y lo publicó en el dashboard. la alarma se activó como se esperaba y el histórico quedó registrado sin pérdidas de información.

El principal hallazgo fue un retardo perceptible en el envío a través del módulo wifi; la actualización del dashboard tardó más de lo deseado por mucha latencia. La medición local e I2C se mantuvo rápida y consistente; el retraso provino del módulo wifi. para mitigarlo se amplió el intervalo de publicación con el fin de evitar la congestión.

En el siguiente video se muestra el sistema funcionando:
link
# Documentación del Código
```
/*
  -----------------------------------------------
  Nodo Periférico I²C - Lectura de Humedad (simulada)
  -----------------------------------------------
  Rol:
    - Actúa como ESCLAVO I²C (addr = 0x08).
    - Lee un valor analógico en A0 (0..1023) como “humedad simulada”.
    - Convierte a % de humedad (0..100) y lo entrega al maestro cuando se lo solicita.

  Hardware:
    - SENSOR_PIN (A0): conectar a salida analógica del sensor o potenciómetro.
    - I²C: SDA/SCL del microcontrolador + resistencias pull-up (si tu placa no las trae).
    - Dirección I²C: 0x08.

  Protocolo:
    - El maestro hace un request (I²C read). Este esclavo responde con 1 byte (0..100).

  Nota:
    - Este ejemplo “invierte” la lectura (map 0→100, 1023→0). Útil para sensores
      donde valor analógico bajo = humedad alta. Ajusta el map si tu sensor es directo.

  Autor: (tu nombre/equipo)
  Fecha: (AAAA-MM-DD)
*/

#include <Wire.h>

#define SENSOR_PIN  A0     // Entrada analógica del sensor/potenciómetro
#define SLAVE_ADDR  0x08   // Dirección I²C de este esclavo

volatile uint8_t humidityValue = 0; // % de humedad (0..100). uint8_t = 1 byte

// --- Prototipo del callback que atiende la petición I²C del maestro ---
void requestEvent();

void setup() {
  // Inicia el bus I²C como ESCLAVO con la dirección indicada
  Wire.begin(SLAVE_ADDR);

  // Registra la función que se llamará automáticamente
  // cuando el maestro solicite datos (I²C read)
  Wire.onRequest(requestEvent);

  // Consola para diagnóstico (opcional)
  Serial.begin(9600);
  Serial.println(F("I2C Slave listo. Addr=0x08"));
}

void loop() {
  // 1) Leer valor analógico crudo (0..1023)
  int raw = analogRead(SENSOR_PIN);

  // 2) Convertir a % de humedad (0..100)
  //    Mapeo invertido: 0 -> 100% (humedad alta), 1023 -> 0% (humedad baja)
  int pct = map(raw, 0, 1023, 100, 0);

  // 3) Limitar al rango válido (seguridad)
  if (pct < 0)   pct = 0;
  if (pct > 100) pct = 100;

  // 4) Guardar en variable compartida (1 byte)
  humidityValue = static_cast<uint8_t>(pct);

  // 5) Log de diagnóstico
  Serial.print(F("Humedad medida: "));
  Serial.print(humidityValue);
  Serial.println(F("%"));

  // 6) Periodo de muestreo (ajusta según necesidad)
  delay(1000);
}

// ---------------------------------------------------------------------
// Callback I²C: se ejecuta AUTOMÁTICAMENTE cuando el maestro pide datos
// Enviamos 1 byte (0..100) con el % de humedad.
// ---------------------------------------------------------------------
void requestEvent() {
  Wire.write(humidityValue); // Respuesta inmediata al maestro
}
```
