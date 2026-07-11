# 85 - MQTT JSON Payload publisher (serialize DHT22 values)

Configure the ESP32 as an MQTT telemetry node, read temperature and humidity levels from a DHT22 sensor on GPIO 4, serialize the measurements into a JSON string using the ArduinoJson library, and publish the packet to the topic `esp32/sensor/climate` every 5 seconds.

## Goal
Learn how to compile sensor payloads using `ArduinoJson`, serialize JSON documents into text buffers, manage publish intervals, and design structured telemetry nodes.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A DHT22 sensor is on GPIO 4. The ESP32 connects to the public HiveMQ broker. Every 5 seconds, it reads the temperature and humidity, compiles a JSON package (e.g. `{"temp":24.5,"humidity":45.0,"status":"OK"}`), and publishes the string to the topic `esp32/sensor/climate`.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | DATA | GPIO4 | Yellow | Temp & Humidity input |
| Shared power | VCC / GND | 3V3 / GND | Red / Black | Sensor power rails |

> **Wiring tip:** Connect the DHT22 data pin directly to GPIO 4.

## Code
```cpp
// MQTT JSON Payload publisher (Structured Climate Telemetry Node)
#include <WiFi.h>
#include <PubSubClient.h>
#include <DHT.h>
#include <ArduinoJson.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const char* mqttServer = "broker.hivemq.com";
const int mqttPort = 1883;

// Telemetry topic
const char* telemetryTopic = "esp32/sensor/climate";

const int DHT_PIN = 4;
#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

WiFiClient espClient;
PubSubClient client(espClient);

unsigned long lastPublishTime = 0;
const unsigned long PUBLISH_INTERVAL_MS = 5000; // Publish telemetry every 5 seconds

void callback(char* topic, byte* payload, unsigned int length) {
  // Callback not required for publish-only node
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    String clientId = "ESP32Client-" + String(random(0xffff), HEX);
    
    if (client.connect(clientId.c_str())) {
      Serial.println("connected.");
      client.publish("esp32/status", "JSON_PUBLISHER_ACTIVE");
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" | Retrying in 5 seconds...");
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  dht.begin();
  
  Serial.println("\nESP32 MQTT JSON Publisher Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  client.setServer(mqttServer, mqttPort);
  client.setCallback(callback);
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();
  
  // 1. Maintain non-blocking publish interval (every 5 seconds)
  unsigned long now = millis();
  if (now - lastPublishTime >= PUBLISH_INTERVAL_MS) {
    float temp = dht.readTemperature();
    float hum = dht.readHumidity();
    
    if (!isnan(temp) && !isnan(hum)) {
      // 2. Allocate StaticJsonDocument buffer (256 bytes is sufficient)
      StaticJsonDocument<256> doc;
      
      // 3. Populate JSON document tree
      doc["temp"] = serialized(String(temp, 1));       // Store with 1 decimal place
      doc["humidity"] = serialized(String(hum, 1));   // Store with 1 decimal place
      doc["status"] = (temp > 35.0) ? "OVERHEAT" : "NORMAL";
      doc["uptime_s"] = now / 1000;
      
      // 4. Serialize JSON document to a text string buffer
      char jsonBuffer[256];
      serializeJson(doc, jsonBuffer);
      
      // 5. Publish serialized JSON to MQTT topic
      bool success = client.publish(telemetryTopic, jsonBuffer);
      
      if (success) {
        Serial.print("[MQTT JSON Publish] Sent -> Topic: esp32/sensor/climate | Payload: ");
        Serial.println(jsonBuffer);
      } else {
        Serial.println("[MQTT Error] JSON Publish failed!");
      }
    } else {
      Serial.println("[Sensor Error] Failed to read from DHT22!");
    }
    
    lastPublishTime = now;
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **DHT22** onto the canvas.
2. Wire DHT22 to **GPIO4**.
3. Paste the code and click **Run**.
4. Adjust the DHT22 temperature and humidity sliders. Verify that the published JSON payloads log in the Serial Monitor.

## Expected Output
Serial Monitor:
```
ESP32 MQTT JSON Publisher Starting...
....
WiFi Connected.
Attempting MQTT connection...connected.
[MQTT JSON Publish] Sent -> Topic: esp32/sensor/climate | Payload: {"temp":24.5,"humidity":45.0,"status":"NORMAL","uptime_s":5}
[MQTT JSON Publish] Sent -> Topic: esp32/sensor/climate | Payload: {"temp":36.2,"humidity":40.0,"status":"OVERHEAT","uptime_s":10}
```

## Expected Canvas Behavior
* Adjusting the DHT22 sliders updates the serialized values in the JSON payload published to the broker, logged to the Serial Monitor every 5 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `StaticJsonDocument<256> doc` | Allocates memory on the stack to compile the JSON tree structure. |
| `doc["temp"] = ...` | Assigns key-value pairs to the document tree. |
| `serializeJson(doc, jsonBuffer)` | Converts the document structure into a serialized string buffer. |
| `client.publish(..., jsonBuffer)` | Transmits the serialized JSON string to the broker. |

## Hardware & Safety Concept: Structured JSON vs Multiple Topics
When a device needs to send multiple values (like temperature, humidity, and status), it can either:
1. Publish values to separate topics (e.g. `temp`, `humidity`, and `status`). This requires establishing three separate publish cycles, increasing network traffic.
2. Serialize values into a single **JSON payload** and publish it to a single topic. This packages all values in a single packet, reducing bandwidth and ensuring the readings are synchronized at the receiver.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the last JSON values published.
2. **Add RSSI to Payload**: Add a key `rssi` to the JSON document and include the WiFi signal strength.
3. **Buzzer confirmation tone**: Sound a quick beep on a buzzer (GPIO 15) when a transmission completes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Deserialization error on subscriber | JSON buffer overflow | Increase the JSON buffer size (e.g. from 256 to 512 bytes) if adding more parameters |
| values display as null on subscriber | Typo in key names | Confirm that the subscriber parses keys matching the published JSON |
| Sockets drop frequently | Keepalive ping missing | Ensure `client.loop()` is called continuously in the loop function |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [38 - ESP32 HTTP JSON API Parser](../beginner/38-esp32-http-json-api-parser.md)
- [49 - ESP32 Wireless DHT22 logger](../beginner/49-esp32-wireless-dht22-logger.md)
- [86 - ESP32 MQTT JSON Payload subscriber](86-esp32-mqtt-json-payload-subscriber-deserialize-commands.md) (Next project)
