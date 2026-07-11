# 80 - MQTT Two-Way control broker link

Configure a unified two-way MQTT node on the ESP32 that establishes a persistent broker connection, publishes raw potentiometer telemetry to `esp32/telemetry/pot` every 5 seconds, subscribes to command topic `esp32/control/led` to switch an LED on GPIO 13, and publishes confirmation feedback.

## Goal
Learn how to implement two-way MQTT client nodes, publish telemetry data, subscribe to control channels, handle callbacks, and confirm output status changes.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and establishes a persistent connection to the HiveMQ broker. A potentiometer is on GPIO 34, and an LED on GPIO 13. The ESP32 publishes the potentiometer value to `esp32/telemetry/pot` every 5 seconds. Simultaneously, it subscribes to `esp32/control/led` to turn the LED ON and OFF, and publishes a status confirmation payload (e.g. `LED_STATE:ON`) to `esp32/status/led`.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Wiper | GPIO34 | Yellow | Analog sensor signal |
| Red LED | Anode (+) | GPIO13 via 330 Ω | Red | Controlled output LED |
| Shared ground | GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the Red LED anode to GPIO 13 via a 330 Ω resistor. Connect the potentiometer wiper to GPIO 34.

## Code
```cpp
// MQTT Two-Way control broker link (Unified Telemetry & Actuation Node)
#include <WiFi.h>
#include <PubSubClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const char* mqttServer = "broker.hivemq.com";
const int mqttPort = 1883;

// Configure MQTT Topics
const char* publishTelemetryTopic = "esp32/telemetry/pot";
const char* subscribeCommandTopic = "esp32/control/led";
const char* publishFeedbackTopic   = "esp32/status/led";

const int POT_PIN = 34;
const int LED_PIN = 13;

WiFiClient espClient;
PubSubClient client(espClient);

unsigned long lastPublishTime = 0;
const unsigned long PUBLISH_INTERVAL_MS = 5000; // Publish telemetry every 5 seconds

// 1. Unified Callback Handler for commands
void callback(char* topic, byte* payload, unsigned int length) {
  // Convert payload array to string
  String message = "";
  for (int i = 0; i < length; i++) {
    message += (char)payload[i];
  }
  message.trim();
  
  Serial.printf("[MQTT Sub] Received Command -> Topic: %s | Payload: %s\n", topic, message.c_str());
  
  // Verify command topic
  if (String(topic) == subscribeCommandTopic) {
    if (message == "ON") {
      digitalWrite(LED_PIN, HIGH);
      Serial.println("[Action] LED turned ON.");
      
      // 2. Publish confirmation feedback
      client.publish(publishFeedbackTopic, "LED_STATE:ON");
    } 
    else if (message == "OFF") {
      digitalWrite(LED_PIN, LOW);
      Serial.println("[Action] LED turned OFF.");
      
      // Publish confirmation feedback
      client.publish(publishFeedbackTopic, "LED_STATE:OFF");
    } 
    else {
      Serial.println("[Error] Unknown command payload!");
    }
  }
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    String clientId = "ESP32Client-" + String(random(0xffff), HEX);
    
    if (client.connect(clientId.c_str())) {
      Serial.println("connected.");
      
      // 3. Subscribe to command channel on reconnection
      client.subscribe(subscribeCommandTopic);
      Serial.printf("Subscribed to topic: %s\n", subscribeCommandTopic);
      
      // Publish node status
      client.publish("esp32/status", "TWO_WAY_NODE_ACTIVE");
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
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW); // Start OFF
  
  Serial.println("\nESP32 Two-Way MQTT Client Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  client.setServer(mqttServer, mqttPort);
  client.setCallback(callback); // Register callback function
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();
  
  // 4. Periodically publish sensor telemetry (every 5 seconds)
  unsigned long now = millis();
  if (now - lastPublishTime >= PUBLISH_INTERVAL_MS) {
    int potVal = analogRead(POT_PIN);
    String payload = String(potVal);
    
    bool success = client.publish(publishTelemetryTopic, payload.c_str());
    
    if (success) {
      Serial.print("[MQTT Pub] Sent -> Topic: esp32/telemetry/pot | Payload: ");
      Serial.println(payload);
    }
    
    lastPublishTime = now;
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer**, and **LED** onto the canvas.
2. Wire Potentiometer to **GPIO34** and LED to **GPIO13**.
3. Paste the code and click **Run**.
4. Slide the potentiometer widget. Verify that telemetry publishes every 5 seconds.
5. Publish the command `ON` or `OFF` to `esp32/control/led` from any MQTT client utility. Verify that the LED toggles and confirmation feedback is published to `esp32/status/led`.

## Expected Output
Serial Monitor:
```
ESP32 Two-Way MQTT Client Starting...
....
WiFi Connected.
Attempting MQTT connection...connected.
Subscribed to topic: esp32/control/led
[MQTT Pub] Sent -> Topic: esp32/telemetry/pot | Payload: 2048
[MQTT Sub] Received Command -> Topic: esp32/control/led | Payload: ON
[Action] LED turned ON.
```

## Expected Canvas Behavior
* Adjusting the potentiometer updates the published telemetry data logged in the Serial Monitor.
* Publishing command values toggles the Red LED widget, logging events in the Serial Monitor.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `client.subscribe(...)` | Registers the subscription filter on the broker. |
| `client.publish(...)` | Packages the data string and transmits it to the MQTT broker under the specified topic. |
| `callback(...)` | Executed when a message matching the subscription arrives. |
| `client.loop()` | Processes background frames to parse subscription events. |

## Hardware & Safety Concept: Closed-Loop Controls (Feedback Status)
In remote control systems, confirmation feedback is essential. If the controller only subscribes to commands, the remote dashboard has no way of verifying if the command was actually executed (e.g. if the LED is broken or the pin failed). Enforcing **closed-loop feedback** by publishing a status confirmation payload ensures the dashboard displays the actual hardware state.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display both the potentiometer value and the LED status.
2. **Double channel switch**: Add a second LED on GPIO 12 and subscribe to a separate topic `esp32/control/led2` to control it.
3. **Buzzer confirmation tone**: Sound a quick beep on a buzzer (GPIO 15) when the LED state changes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Commands are ignored | Callback not registered | Ensure `client.setCallback(callback)` is called in setup |
| Telemetry fails to publish | Connection offline | Verify that the client is connected to the broker before publishing |
| Conflicting status logs | Client ID conflict | Ensure you generate a unique Client ID for each connection |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [76 - ESP32 MQTT Client Connection](76-esp32-mqtt-client-connection.md)
- [78 - ESP32 MQTT Topic publisher (Sensor telemetry stream)](78-esp32-mqtt-topic-publisher-sensor-telemetry-stream.md)
- [79 - ESP32 MQTT Topic subscriber (Remote pin control command)](79-esp32-mqtt-topic-subscriber-remote-pin-control-command.md)
