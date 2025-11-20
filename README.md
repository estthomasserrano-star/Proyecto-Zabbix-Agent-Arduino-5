# Proyecto-Zabbix-Agent-Arduino-5
# Arduino-Zabbix-Agent
Se desarrollará un proyecto que integra una placa Arduino Uno junto con un módulo Ethernet W5100, con el propósito de realizar el monitoreo de diferentes sensores mediante la plataforma Zabbix. 
Este sistema permitirá la recolección y envío de datos en tiempo real, facilitando la supervisión del estado de las variables medidas y la generación de alertas automáticas ante cualquier condición anómala detectada.
Correcta ejecuccion y monitoreo de alertas correctamente 
 en este proyecto tendremos los siguientes items 

* Materiales](#materiales)
* [Esquematico](#)
   sensores Implementados
* [DHT11](#dht11)
 * [DS18b20](#ds18b20)
 * [PIR](#pir)

* [Conclusiones](#Conclusiones)
* [Como Usar](Como usar)

* [en el codigo](en el codigo)
* [Ejemplo](#ejemplo)

Materiales 
- Arduino Uno v3
- Ethernet Shield W5100
- Arduino Sensor Shield*
- DHT11 (o similar) y una resistencia de 10k
- DS18B20 (o similar) y una resistencia de 4.7k
- PIR (sensor infrarrojo pasivo / detector de movimiento)
- Un LED y una resistencia de 1k
Esquematico



Informacion Relevante sensores implemtados
#Sensor PIR (Movimiento)
detectar los cambios en esta radiación. Cuando un objeto, como una persona o un animal, se mueve dentro del área de detección.

#Sensor DHT11(Temperatura)
El DHT11 es un sensor digital que permite medir temperatura y humedad relativa del ambiente
-Mide temperatura de 0 °C a 50 °C con una precisión aproximada de ±2 °C.
-Mide humedad relativa de 20 % a 90 % con una precisión de ±5 %.

#Sensor DS18B20
s un sensor digital de temperatura de alta precisión que utiliza el protocolo 1-Wire, lo que permite que varios sensores se conecten en un mismo pin de datos 
sin necesidad de múltiples entradas

-Rango de medición: –55 °C a +125 °C.

-Precisión:
±0.5 °C en el rango de –10 °C a +85 °C

##Conclusiones 
-Este proyecto permitió evidenciar cómo los sensores pueden integrarse de manera efectiva en nuestro entorno cotidiano, tanto en el ámbito personal como en el empresarial. A través de la implementación de diferentes dispositivos de medición y monitoreo, se demostró cómo estas tecnologías contribuyen a mejorar procesos, 
optimizar recursos y fortalecer la toma de decisiones en organizaciones
-el correcto funcionamiento de envio de alertas generados por zabbix un aherramienta util de monitorizacion

#Funcionamiento
#Codigo
/*
 * ARDUINO ACTIVE HTTP AGENT (ESTILO ZABBIX-AGENT)
 * Adaptado para Thomas Serrano
 *
 * Este agente:
 *   - Lee sensores DHT11, DS18B20 y PIR
 *   - Mantiene valores durante X segundos para evitar oscilaciones
 *   - Usa claves al estilo Zabbix: q, w, e, r, t, etc.
 *   - Construye una URL HTTP GET con esa clave
 *   - Envía los datos al collector 172.28.160.10
 *
 * NOTA IMPORTANTE:
 * No es un Zabbix-Agent real. Es un “Active Client” con la
 * arquitectura del proyecto original de Arduino-Zabbix-Agent.
 */

#include <SPI.h>
#include <Ethernet.h>
#include <DHT.h>
#include <OneWire.h>
#include <DallasTemperature.h>

// ---------------------------------------------------------------------------
// 1. CONFIGURACIÓN DE PINES
// ---------------------------------------------------------------------------
#define LED_PIN       13   // LED indicador, tipo Zabbix-Agent original
#define DHT_PIN       6
#define PIR_PIN       10
#define ONE_WIRE_BUS  7
#define DHT_TYPE      DHT11

#define SOIL_PIN      A0   // NO LO USAS, PERO LO AGREGO POR ESTRUCTURA ZABBIX

// ---------------------------------------------------------------------------
// 2. OBJETOS DE SENSORES
// ---------------------------------------------------------------------------
DHT dht(DHT_PIN, DHT_TYPE);
OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature ds18b20(&oneWire);

// ---------------------------------------------------------------------------
// 3. CONFIGURACIÓN DE RED
// ---------------------------------------------------------------------------
byte mac[] = { 0xDE, 0xAD, 0xBE, 0xEF, 0xFE, 0x05 };
IPAddress ip(172,28,160,1);
IPAddress gateway(172,28,160,254);
IPAddress subnet(255,255,255,0);

IPAddress collectorIP(172,28,160,10);
uint16_t collectorPort = 80;

EthernetClient client;

// ---------------------------------------------------------------------------
// 4. VARIABLES DE SENSORES con retención de valores (ESTILO ZABBIX)
// ---------------------------------------------------------------------------

// Tiempo de retención de sensores (evita lecturas rápidas desestables)
unsigned long lastReadDHT     = 0;
unsigned long lastReadDS      = 0;
unsigned long lastReadPIR     = 0;

const unsigned long DHT_INTERVAL = 5000;    // 5s
const unsigned long DS_INTERVAL  = 15000;   // 15s
const unsigned long PIR_INTERVAL = 2000;    // 2s

float lastTempDHT = 0;
float lastHumDHT  = 0;
float lastTempDS  = 0;
int   lastPIR     = 0;

// ---------------------------------------------------------------------------
// 5. FUNCIONES DE LECTURA (ESTILO ZABBIX-AGENT ORIGINAL)
// ---------------------------------------------------------------------------

// --- SENSOR: Humedad del suelo (aunque no lo uses) ---
int readSoil() {
  return digitalRead(SOIL_PIN);
}

// --- SENSOR: DHT11 ---
void readDHT() {
  if (millis() - lastReadDHT >= DHT_INTERVAL) {
    float h = dht.readHumidity();
    float t = dht.readTemperature();

    if (!isnan(h))  lastHumDHT = h;
    if (!isnan(t))  lastTempDHT = t;

    lastReadDHT = millis();
  }
}

// --- SENSOR: DS18B20 ---
void readDS() {
  if (millis() - lastReadDS >= DS_INTERVAL) {
    ds18b20.requestTemperatures();
    float tC = ds18b20.getTempCByIndex(0);

    if (tC != DEVICE_DISCONNECTED_C)
      lastTempDS = tC;

    lastReadDS = millis();
  }
}

// --- SENSOR: PIR ---
void readPIR() {
  if (millis() - lastReadPIR >= PIR_INTERVAL) {
    lastPIR = digitalRead(PIR_PIN);
    digitalWrite(LED_PIN, lastPIR);
    lastReadPIR = millis();
  }
}

// ---------------------------------------------------------------------------
// 6. PARSEADOR DE CLAVES (MUY SIMILAR AL ZABBIX-AGENT)
// ---------------------------------------------------------------------------
String readKey(char k) {

  switch (k) {

    case 'q':   // Humedad de suelo
      return String(readSoil());

    case 'w':   // Temp DHT11
      return String(lastTempDHT, 1);

    case 'e':   // Humedad DHT11
      return String(lastHumDHT, 1);

    case 'r':   // Temp DS18B20
      return String(lastTempDS, 1);

    case 't':   // PIR
      return String(lastPIR);

    default:
      return "unknown";
  }
}

// ---------------------------------------------------------------------------
// 7. ENVÍO DEL DATO AL COLLECTOR (HTTP GET ACTIVO)
// ---------------------------------------------------------------------------
void sendToCollector(char key) {

  String value = readKey(key);

  Serial.print("[TX] clave: ");
  Serial.print(key);
  Serial.print(" valor=");
  Serial.println(value);

  if (client.connect(collectorIP, collectorPort)) {
    
    String url = "/recv?host=arduino01";
    url += "&key=";
    url += key;
    url += "&value=";
    url += value;

    client.print(String("GET ") + url + " HTTP/1.1\r\n" +
                 "Host: 172.28.160.10\r\n" +
                 "Connection: close\r\n\r\n");

    Serial.print("URL enviada: ");
    Serial.println(url);

    unsigned long t = millis();
    while (client.connected() && millis() - t < 1200) {
      if (client.available()) client.read();
    }
    client.stop();
  }
  else {
    Serial.println("ERROR: No se pudo conectar");
  }
}

// ---------------------------------------------------------------------------
// 8. SETUP
// ---------------------------------------------------------------------------
void setup() {
  Serial.begin(9600);

  pinMode(LED_PIN, OUTPUT);
  pinMode(PIR_PIN, INPUT);

  dht.begin();
  ds18b20.begin();

  Ethernet.begin(mac, ip, gateway, gateway, subnet);

  delay(1000);
  Serial.print("IP: ");
  Serial.println(Ethernet.localIP());
}

// ---------------------------------------------------------------------------
// 9. LOOP PRINCIPAL (ESTILO ZABBIX – CICLO SENSORES → ENVÍO)
// ---------------------------------------------------------------------------
void loop() {

  // 1. Mantener sensores actualizados
  readDHT();
  readDS();
  readPIR();

  // 2. Enviar cada clave (puedes activar/desactivar)
  sendToCollector('w');   // Temp DHT
  sendToCollector('e');   // Humedad DHT
  sendToCollector('r');   // Temp DS18B20
  sendToCollector('t');   // PIR

  delay(2000);  // Igual que tu código
}
Agregar Items en el servidor Zabbix.

Configuracion Items Zabbix 
e – Humedad del aire medida por el sensor DHT11.

r – Temperatura del aire medida por el DS18B20 cuyo número de serie termina en 17.

f – Temperatura del aire medida por el DS18B20 cuyo número de serie termina en B6.

v – Temperatura del aire medida por el DS18B20 cuyo número de serie termina en D3.

t – Sensor de movimiento PIR.
## En el codigo
se realiza dicha configuracion en el codigo con dirreccion Ip,puerta de enlace, direcccion MAC utilizada Agente

-IPAddress ip(172,28,160,1);                      // IP fija del Arduino
-IPAddress gateway(172,28,160,254);               // Puerta de enlace
-IPAddress subnet(255,255,255,0);                 // Máscara de subred

#Configuracion De Triggers
Esta configuración tiene como finalidad generar la alerta correspondiente en el momento en que alguno de los sensores detecte una condición específica, 
permitiendo así una respuesta rápida ante cualquier evento relevante.
