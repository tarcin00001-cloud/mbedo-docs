# 79 - MQTT Topic subscriber (Remote pin control command)

Configure the ESP32 as an MQTT subscriber node, register a callback handler, subscribe to the command topic `esp32/control/led`, and switch a local LED on GPIO 13 when commands ("ON", "OFF") are received.

## Goal
Learn how to subscribe to MQTT topics, parse incoming payload arrays, write pin states from callbacks, and handle connection drops.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An LED is on GPIO 13. The ESP32 connects to the public HiveMQ broker and subscribes to the topic `esp32/control/led`. Publishing the command `ON` or `OFF` to this topic from any MQTT client utility controls the LED immediately.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Red LED | Anode (+) | GPIO13 via 330 Ω | Red | Controlled output LED |
| Shared ground | GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the Red LED anode to GPIO 13 via a 330 Ω resistor.

## Code
```cpp
// MQTT Topic subscriber (Remote LED Controller Node)
#include <WiFi.h>
#include <PubSubClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const char* mqttServer = "broker.hivemq.com";
const int mqttPort = 1883;

// Topic to subscribe to
const char* commandTopic = "esp32/control/led";

const int LED_PIN = 13;

WiFiClient espClient;
PubSubClient client(espClient);

// 1. MQTT Callback Handler
// Automatically executed when a message arrives on subscribed topics
void callback(char* topic, byte* payload, unsigned int length) {
  // Convert payload array to string
  String message = "";
  for (int i = 0; i < length; i++) {
    message += (char)payload[i];
  }
  message.trim();
  
  Serial.printf("[MQTT Sub] Message arrived -> Topic: %s | Payload: %s\n", topic, message.c_str());
  
  // 2. Evaluate command
  if (String(topic) == commandTopic) {
    if (message == "ON") {
      digitalWrite(LED_PIN, HIGH);
      Serial.println("[Action] LED turned ON.");
    } 
    else if (message == "OFF") {
      digitalWrite(LED_PIN, LOW);
      Serial.println("[Action] LED turned OFF.");
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
      
      // 3. Subscribe to target topic
      // Must be called after every successful connection to register on the broker
      bool success = client.subscribe(commandTopic);
      
      if (success) {
        Serial.printf("Subscribed to topic: %s\n", commandTopic);
      } else {
        Serial.println("Failed to subscribe!");
      }
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
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW); // Start OFF
  
  Serial.println("\nESP32 MQTT Subscriber Starting...");
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
  
  // 4. Yield background processing to handle subscription events
  client.loop();
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Wire LED to **GPIO13**.
3. Paste the code and click **Run**.
4. In simulation, use an MQTT client utility (or public MQTT dashboard) to publish the command `ON` or `OFF` to the topic `esp32/control/led`. Watch the LED state change.

## Expected Output
Serial Monitor:
```
ESP32 MQTT Subscriber Starting...
....
WiFi Connected.
Attempting MQTT connection...connected.
Subscribed to topic: esp32/control/led
[MQTT Sub] Message arrived -> Topic: esp32/control/led | Payload: ON
[Action] LED turned ON.
[MQTT Sub] Message arrived -> Topic: esp32/control/led | Payload: OFF
[Action] LED turned OFF.
```

## Expected Canvas Behavior
* Publishing `ON` or `OFF` to the topic switches the Red LED widget ON and OFF.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `client.subscribe(...)` | Registers the subscription filter on the broker. |
| `callback(...)` | Executed when a message matching the subscription arrives. |
| `client.loop()` | Processes background frames to parse subscription events. |

## Hardware & Safety Concept: Re-Subscription on Reconnection
MQTT subscriptions are stored in the broker's active session. If the connection drops (due to WiFi drop out or keepalive timeout), the broker discards the session. When the client reconnects, it must call `client.subscribe()` to register the subscription again. Placing the subscription call inside the `reconnect()` function ensures the device remains controllable after reconnecting.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a checklist showing the last command received.
2. **Double Channel Receiver**: Add a second LED on GPIO 12 and subscribe to a separate topic `esp32/control/led2` to control it.
3. **Buzzer confirmation tone**: Sound a quick beep on a buzzer (GPIO 15) when the LED state changes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Commands are ignored | Callback not registered | Ensure `client.setCallback(callback)` is called in setup |
| Commands have no effect | Topic mismatch | Verify that the command is published to the exact same topic (`esp32/control/led`) |
| The ESP32 does not receive commands | `client.loop()` missing | Verify that `client.loop()` is called continuously in the loop function |

## Mode Notes
Projec runs in MbedO **interpreted C++ mode**.

## Related Projects
- [21 - ESP32 UDP Control Switch](../beginner/21-esp32-udp-control-switch.md)
- [76 - ESP32 MQTT Client Connection](76-esp32-mqtt-client-connection.md)
- [80 - ESP32 MQTT Two-Way control broker link](80-esp32-mqtt-two-way-control-broker-link.md) (Next project)
