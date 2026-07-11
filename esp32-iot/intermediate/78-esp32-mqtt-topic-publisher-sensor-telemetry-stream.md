# 78 - MQTT Topic publisher (Sensor telemetry stream)

Configure the ESP32 as an MQTT telemetry publisher, read an analog potentiometer sensor on GPIO 34, package the values into a text payload, and publish the data to the topic `esp32/sensor/potentiometer` every 2 seconds.

## Goal
Learn how to compile sensor payloads, publish messages to MQTT topics, manage transmit intervals, and monitor connection states.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A potentiometer is on GPIO 34. The ESP32 connects to the public HiveMQ broker. Every 2 seconds, it reads the potentiometer and publishes the raw value to the topic `esp32/sensor/potentiometer`.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Wiper | GPIO34 | Yellow | Analog sensor signal |
| Shared power | VCC / GND | 3V3 / GND | Red / Black | Sensor power rails |

> **Wiring tip:** Connect the potentiometer wiper to GPIO 34.

## Code
```cpp
// MQTT Topic publisher (Potentiometer Telemetry Streamer)
#include <WiFi.h>
#include <PubSubClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const char* mqttServer = "broker.hivemq.com";
const int mqttPort = 1883;

const int POT_PIN = 34;

WiFiClient espClient;
PubSubClient client(espClient);

unsigned long lastPublishTime = 0;
const unsigned long PUBLISH_INTERVAL_MS = 2000; // Publish telemetry every 2 seconds

void callback(char* topic, byte* payload, unsigned int length) {
  // Callback not required for publish-only node
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    String clientId = "ESP32Client-" + String(random(0xffff), HEX);
    
    if (client.connect(clientId.c_str())) {
      Serial.println("connected.");
      client.publish("esp32/status", "TELEMETRY_NODE_ACTIVE");
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
  
  pinMode(POT_PIN, INPUT);
  
  Serial.println("\nESP32 MQTT Publisher Starting...");
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
  
  // 1. Maintain non-blocking publish interval
  unsigned long now = millis();
  if (now - lastPublishTime >= PUBLISH_INTERVAL_MS) {
    // 2. Read analog potentiometer value
    int potVal = analogRead(POT_PIN);
    
    // Convert value to string payload
    String payload = String(potVal);
    
    // 3. Publish to MQTT topic
    // Topic: "esp32/sensor/potentiometer"
    bool success = client.publish("esp32/sensor/potentiometer", payload.c_str());
    
    if (success) {
      Serial.print("[MQTT Publish] Sent -> Topic: esp32/sensor/potentiometer | Payload: ");
      Serial.println(payload);
    } else {
      Serial.println("[MQTT Error] Publish failed!");
    }
    
    lastPublishTime = now;
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Potentiometer** onto the canvas.
2. Wire Potentiometer to **GPIO34**.
3. Paste the code and click **Run**.
4. Slide the potentiometer widget. Verify that the published values log in the Serial Monitor.
5. In hardware, use an MQTT client utility (e.g. MQTT.fx or broker dashboard) to subscribe to the topic and watch the values.

## Expected Output
Serial Monitor:
```
ESP32 MQTT Publisher Starting...
....
WiFi Connected.
Attempting MQTT connection...connected.
[MQTT Publish] Sent -> Topic: esp32/sensor/potentiometer | Payload: 2048
[MQTT Publish] Sent -> Topic: esp32/sensor/potentiometer | Payload: 3100
```

## Expected Canvas Behavior
* Adjusting the potentiometer updates the payload published to the broker, logged to the Serial Monitor every 2 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `analogRead(POT_PIN)` | Samples the analog voltage on the potentiometer wiper pin. |
| `client.publish(...)` | Packages the data string and transmits it to the MQTT broker under the specified topic. |
| `now - lastPublishTime >= 2000` | Controls the 0.5 Hz publish rate (2000 ms intervals) to avoid network congestion. |

## Hardware & Safety Concept: MQTT Publish Rates and Network Bandwidth
MQTT is designed to be lightweight, but transmitting high-frequency updates (e.g. 50 Hz) over the internet can saturate cellular links or trigger rate limits on public brokers. Selecting a **refresh interval** (e.g. once every 2 seconds or 5 seconds) for slowly changing telemetry values balances responsiveness and bandwidth.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the value last published.
2. **JSON payload format**: Format the published string as a JSON package (e.g. `{"value":2048}`).
3. **Buzzer confirmation tone**: Sound a quick beep on a buzzer (GPIO 15) when a transmission completes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Publish fails | Connection offline | Verify that the client is connected to the broker before publishing |
| Values missing on subscriber side | Topic mismatch | Ensure the subscriber client is listening on the exact same topic (`esp32/sensor/potentiometer`) |
| The ESP32 hangs | Blocking delays | Avoid using blocking `delay()` calls inside the main loop |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [26 - ESP32 TCP Telemetry Broadcaster](../beginner/26-esp32-tcp-telemetry-broadcaster.md)
- [76 - ESP32 MQTT Client Connection](76-esp32-mqtt-client-connection.md)
- [79 - ESP32 MQTT Topic subscriber (Remote pin control command)](79-esp32-mqtt-topic-subscriber-remote-pin-control-command.md) (Next project)
