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

# Pruebas y validación
Para probar la comunicación I2C se realizó un print para visualizar la lectura de humedad en la consola; se revisaron iteradas lecturas teniendo el sensor seco y envuelto en un paño húmedo.

Para probar la conexión con ThingSpeak, primero fue necesario implementar el sensor de humedad. Se utilizó el panel de lectura y se revisó que se mostrará el mismo valor que en consola local.

Para probar la alerta se realizó un trigger por medio de código para poner en estado de alerta al sistema sin necesidad de un sensor. Luego se configuró la variable para guardar el umbral y por medio de una condición se activó el trigger. Primero se imprimió en consola un print de debug para saber si entraba en alerta y luego se conectó con el dashboard de ThingSpeak.
# Resultados
La arquitectura I2C se implementó y funcionó correctamente; el Arduino esclavo midió el porcentaje de humedad de forma estable y el Arduino maestro recibió el dato, evaluó la alerta y lo publicó en el dashboard. la alarma se activó como se esperaba y el histórico quedó registrado sin pérdidas de información.

El principal hallazgo fue un retardo perceptible en el envío a través del módulo wifi; la actualización del dashboard tardó más de lo deseado por mucha latencia. La medición local e I2C se mantuvo rápida y consistente; el retraso provino del módulo wifi. para mitigarlo se amplió el intervalo de publicación con el fin de evitar la congestión.

En el siguiente video se muestra el sistema funcionando:
link
# Documentación del Código
## Script Esclavo
```
/*
  -----------------------------------------------
  Nodo Periférico I²C - Lectura de Humedad
  -----------------------------------------------
  Rol:
    - Actúa como ESCLAVO I²C (addr = 0x08).
    - Lee un valor analógico como “humedad simulada”.
    - Convierte a % de humedad y lo entrega al maestro cuando se lo solicita.

  Hardware:
    - SENSOR_PIN: conectar a salida analógica del sensor o potenciómetro.
    - I²C: SDA/SCL del microcontrolador + resistencias pull-up.
    - Dirección I²C: 0x08.

  Protocolo:
    - El maestro hace un request (I²C read). Este esclavo responde con 1 byte (0..100).
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

## Script Esclavo
```
/*
  -------------------------------------------------------------
  Controlador (I2C Master) + Uplink IoT (ThingSpeak vía ESP-01)
  -------------------------------------------------------------
  Rol:
    - Lee 1 byte de humedad (%) desde un ESCLAVO I2C (addr 0x08).
    - Enciende LED si humedad > umbral.
    - Publica la humedad en ThingSpeak (HTTP POST) cada ≥20 s.

  Hardware:
    - I2C Master.
    - I2C Slave.
    - ESP-01 (AT firmware) conectado por SoftwareSerial:
        D2 = RX (del micro) -> TX del ESP-01
        D3 = TX (del micro) -> RX del ESP-01

  Notas:
    - ThingSpeak limita a ~15 s mínimo entre updates. Aquí usamos 20 s.
    - Las credenciales WiFi y apiKey están en texto plano (solo prototipos).
    - Librerías: Wire, SoftwareSerial, WiFiEspAT.

*/

#include <Wire.h>
#include <SoftwareSerial.h>
#include <WiFiEspAT.h>

// -------- Configuración I2C / IO ----------
#define SLAVE_ADDR 0x08        // Esclavo I2C que entrega humedad (1 byte)
#define LED_PIN    13          // LED indicador

// --------- UART para el ESP-01 (AT) --------
SoftwareSerial espSerial(2, 3); // RX (D2), TX (D3)

// --------- WiFi / ThingSpeak ---------------
char ssid[] = "Sebas";
char pass[] = "Hanamaru123";

const char* server = "api.thingspeak.com";
String apiKey = "57ZRJ6EBXJRBM9NE";  // API Key de escritura (Write API Key)

WiFiClient client;

// --------- Timers / Constantes -------------
unsigned long lastRead = 0;
const unsigned long INTERVAL = 20000UL;  // 20 s (≥15 s recomendado por ThingSpeak)
const uint8_t HUM_LED_THRESHOLD = 3;     // Umbral simple para LED

// ---------- Prototipos ----------
bool ensureWiFi();
bool postToThingSpeak(int humidity);
int  readHumidityFromSlave();

// =========================================================
void setup() {
  Serial.begin(9600);
  Wire.begin();                        // I2C Master
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);

  // Iniciar ESP-01 (AT) con WiFiEspAT
  espSerial.begin(115200);
  WiFi.init(&espSerial);

  Serial.println(F("Inicializando modulo WiFi (ESP-01 AT)..."));
  if (WiFi.status() == WL_NO_MODULE) {
    Serial.println(F("ERROR: No se detecta el modulo WiFi."));
    while (true) { delay(1000); }
  }

  // Intento de conexión inicial
  ensureWiFi();
}

// =========================================================
void loop() {
  // Publicación periódica
  if (millis() - lastRead >= INTERVAL) {
    lastRead = millis();

    // (1) Asegurar WiFi conectado
    if (!ensureWiFi()) {
      Serial.println(F("WiFi no disponible. Se reintentara en el siguiente ciclo."));
      return;
    }

    // (2) Leer 1 byte del esclavo I2C
    int humidity = readHumidityFromSlave();
    if (humidity < 0) {  // -1 => sin datos
      Serial.println(F("No hay datos del esclavo."));
      return;
    }

    // (3) Lógica local simple (LED)
    digitalWrite(LED_PIN, (humidity > HUM_LED_THRESHOLD) ? HIGH : LOW);

    // (4) Enviar a ThingSpeak
    if (postToThingSpeak(humidity)) {
      Serial.print(F("Dato enviado a ThingSpeak: "));
      Serial.println(humidity);
    } else {
      Serial.println(F("Error al enviar a ThingSpeak."));
    }
  }
}

// =========================================================
// Asegura que haya conexión WiFi. Reintenta si se perdió.
bool ensureWiFi() {
  if (WiFi.status() == WL_CONNECTED) return true;

  Serial.print(F("Conectando a "));
  Serial.println(ssid);
  WiFi.begin(ssid, pass);

  unsigned long t0 = millis();
  const unsigned long TO = 15000; // 15 s timeout
  while (WiFi.status() != WL_CONNECTED && (millis() - t0) < TO) {
    delay(500);
    Serial.print('.');
  }
  Serial.println();

  if (WiFi.status() == WL_CONNECTED) {
    Serial.print(F("WiFi OK, IP: "));
    Serial.println(WiFi.localIP());
    return true;
  } else {
    Serial.println(F("Timeout de conexion WiFi."));
    return false;
  }
}

// =========================================================
// Lee 1 byte del esclavo I2C (0..100). Devuelve -1 si falla.
int readHumidityFromSlave() {
  Wire.requestFrom(SLAVE_ADDR, 1);
  if (Wire.available() < 1) return -1;

  int h = Wire.read();
  // Sanidad del dato (clamp 0..100)
  if (h < 0)   h = 0;
  if (h > 100) h = 100;

  Serial.print(F("HUM: "));
  Serial.println(h);
  return h;
}

// =========================================================
// Envia la humedad a ThingSpeak por HTTP POST.
// Devuelve true si el servidor responde con "200 OK".
bool postToThingSpeak(int humidity) {
  // Construir payload
  String postStr = "api_key=" + apiKey;
  postStr += "&field1=" + String(humidity);

  // Abrir socket TCP:80
  if (!client.connect(server, 80)) {
    Serial.println(F("No se pudo conectar al servidor."));
    return false;
  }

  // HTTP/1.1 POST
  client.println(F("POST /update HTTP/1.1"));
  client.println(F("Host: api.thingspeak.com"));
  client.println(F("Connection: close"));
  client.println(F("Content-Type: application/x-www-form-urlencoded"));
  client.print  (F("Content-Length: "));
  client.println(postStr.length());
  client.println();
  client.print(postStr);

  // Leer respuesta básica (codigo HTTP)
  bool ok = false;
  unsigned long t0 = millis();
  while (client.connected() && (millis() - t0) < 3000) {
    if (client.available()) {
      String line = client.readStringUntil('\n');
      // Espera cabecera "HTTP/1.1 200 OK"
      if (line.startsWith("HTTP/1.1 200")) ok = true;
      // Fin de cabeceras
      if (line == "\r") break;
    }
  }
  // Consumir cuerpo (opcional)
  while (client.available()) client.read();

  client.stop();
  return ok;
}
```

# Actas
## Acta 1 - Planeación de la implementación del proyecto
* __Proyecto:__ Sistema de monitoreo y publicación IoT (Humedad) con arquitectura I2C
* __Fecha:__ 27/10/2025 — Hora: 2pm - 4pmm — Lugar/Medio: Universidad de La Sabana
* __Miebros:__ Sebastian Rodriguez y Juan Valderrama.
* __Objetivo de la reunión:__ Definir alcance, arquitectura y plan de trabajo inicial.
### Orden del día:
1. Alcance del prototipo y criterios de éxito.
2. Arquitectura de 2 capas (I2C) y flujo de datos a ThingSpeak.
3. Variable a medir (Humedad) y frecuencia de muestreo.
4. Plan de pruebas y evidencias para el informe.
### Desarrollo y acuerdos:
* Alcance: Prototipo funcional que lea Humedad (esclavo) y publique en ThingSpeak (maestro), con alerta basada en umbral.
* Arquitectura: Capa Esclavo I2C (lectura de humedad y filtrado) + Capa Maestro (consulta I2C, lógica de alerta, envío a IoT).
* Variables/frecuencia: Humedad cada 10–30 s; para pruebas extendidas; publicación ≥20 s por límite de ThingSpeak.
* Pruebas: comunicación I2C estable, publicación IoT, activación de alerta y registro histórico.
* Entrega: reporte con diagrama de actividad, código documentado y capturas del dashboard.


* __Cierre:__ Se aprueba el plan y se convoca a reunión de compras/asignación de tareas.

## Acta 2 - Compra de materiales y división del trabajo
* __Proyecto:__ Sistema de monitoreo y publicación IoT (Humedad) con arquitectura I2C
* __Fecha:__ 29/10/2025 — Hora: 9am - 11am — Lugar/Medio: Universidad de La Sabana
* __Miebros:__ Sebastian Rodriguez y Juan Valderrama.
* __Objetivo de la reunión:__ Acordar lista de materiales y lista de tareas.

### Materiales acordados:
* Dos Arduinos modelo MCO-446700374.
* Sensor de humedad de sustrato fc-28.
* Modulo wi-fi esp015.
* Protoboard.

### División del trabajo:
* Compra de materiales.
* Codigo Maestro / Esclavo.
* Dashboard de ThingSpeak.
* Conexión con dashboard de ThingSpeak.
* Conexión fisica de la arquitectura.
* Grabación del video.
* Documentación.

* __Cierre:__ Se aprueban compras y lista de tareas.

## Acta 3 - Elaboración del informe y grabación del video
* __Proyecto:__ Sistema de monitoreo y publicación IoT (Humedad) con arquitectura I2C
* __Fecha:__ 29/10/2025 — Hora: 10am - 1pm — Lugar/Medio: Chia
* __Miebros:__ Sebastian Rodriguez y Juan Valderrama.
* __Objetivo de la reunión:__ Consolidar resultados, preparar informe final y grabar video de demostración.

### Orden del día:
1. Revisión de resultados y evidencias (capturas de ThingSpeak, logs, fotos del montaje).
2. Estructura del informe (contexto, arquitectura I2C, diagrama de actividad, pruebas, resultados, conclusiones).
3. Guion y logística de video demo (escenas y tiempos).
4. Checklist de rúbrica (claridad, trazabilidad, evidencias).

###Lecciones aprendidas:

* La arquitectura I2C es estable; el cuello de botella fue la latencia Wi-Fi.
* Las estrategias de intervalo ≥20s y reintentos mejoraron la confiabilidad.

* __Cierre:__ Se aprueba el guion del video y la estructura final del informe.
