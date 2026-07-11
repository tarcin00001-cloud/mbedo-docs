# 81 - MQTT Subscribe Relay controller

Configure the ESP32 as an MQTT subscriber node, register a callback handler, subscribe to the command topic `esp32/control/relay`, switch a physical relay on GPIO 13 when commands ("OPEN", "CLOSE") are received, and publish status feedback to `esp32/status/relay`.

## Goal
Learn how to subscribe to MQTT topics, parse incoming payload arrays for custom commands, actuate relay contacts, and publish status feedback.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A relay is on GPIO 13. The ESP32 connects to the public HiveMQ broker and subscribes to `esp32/control/relay`. Publishing the command `CLOSE` closes the relay contacts (turns load ON), and publishing `OPEN` opens the contacts (turns load OFF). The ESP32 publishes a confirmation payload (e.g. `RELAY:CLOSED`) to `esp32/status/relay`.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Relay Module | IN (Signal) | GPIO13 | Orange | Load controller switch |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the relay from the 5V Vin rail.

## Code
```cpp
// MQTT Subscribe Relay controller (Secured load switch receiver node)
#include <WiFi.h>
#include <PubSubClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const char* mqttServer = "broker.hivemq.com";
const int mqttPort = 1883;

// Configure MQTT Topics
const char* commandTopic = "esp32/control/relay";
const char* feedbackTopic = "esp32/status/relay";

const int RELAY_PIN = 13;

WiFiClient espClient;
PubSubClient client(espClient);

// MQTT Message Callback Handler
void callback(char* topic, byte* payload, unsigned int length) {
  // Convert payload array to string
  String message = "";
  for (int i = 0; i < length; i++) {
    message += (char)payload[i];
  }
  message.trim();
  
  Serial.printf("[MQTT Sub] Command arrived -> Topic: %s | Payload: %s\n", topic, message.c_str());
  
  // Verify command topic
  if (String(topic) == commandTopic) {
    if (message == "CLOSE") {
      digitalWrite(RELAY_PIN, HIGH); // CLOSE relay contacts (Turn load ON)
      Serial.println("[Action] Relay contacts CLOSED.");
      
      // Publish confirmation feedback
      client.publish(feedbackTopic, "RELAY:CLOSED");
    } 
    else if (message == "OPEN") {
      digitalWrite(RELAY_PIN, LOW); // OPEN relay contacts (Turn load OFF)
      Serial.println("[Action] Relay contacts OPENED.");
      
      // Publish confirmation feedback
      client.publish(feedbackTopic, "RELAY:OPEN");
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
      
      // Subscribe to command channel on reconnection
      client.subscribe(commandTopic);
      Serial.printf("Subscribed to topic: %s\n", commandTopic);
      
      // Publish initial state
      client.publish(feedbackTopic, "RELAY:OPEN");
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
  
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start with relay OPEN (off)
  
  Serial.println("\nESP32 MQTT Relay Subscriber Starting...");
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
  
  // Yield background processing to handle subscription events
  client.loop();
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Relay** onto the canvas.
2. Wire Relay to **GPIO13**.
3. Paste the code and click **Run**.
4. In simulation, publish the command `CLOSE` or `OPEN` to the topic `esp32/control/relay` using an MQTT client utility. Verify that the relay status updates.

## Expected Output
Serial Monitor:
```
ESP32 MQTT Relay Subscriber Starting...
....
WiFi Connected.
Attempting MQTT connection...connected.
Subscribed to topic: esp32/control/relay
[MQTT Sub] Command arrived -> Topic: esp32/control/relay | Payload: CLOSE
[Action] Relay contacts CLOSED.
[MQTT Sub] Command arrived -> Topic: esp32/control/relay | Payload: OPEN
[Action] Relay contacts OPENED.
```

## Expected Canvas Behavior
* Publishing `CLOSE` or `OPEN` to the topic switches the relay widget ON (green) and OFF (grey).

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `client.subscribe(...)` | Registers the subscription filter on the broker. |
| `digitalWrite(RELAY_PIN, ...)` | Controls the relay contacts to switch the connected load. |
| `client.publish(feedbackTopic, ...)` | Publishes a status confirmation payload back to the broker. |
| `client.loop()` | Processes background frames to parse subscription events. |

## Hardware & Safety Concept: Electrical Isolation and Inductive Snubbing
Relays isolate the low-voltage microcontroller (3.3V) from high-voltage AC loads (such as mains power). When a relay switches inductive loads (like motors or transformers), a high-voltage spike is generated across the contacts (back EMF), which can cause arc discharges that melt the contacts. To protect the relay, install an external RC snubber circuit or varistor in parallel with the load.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the active relay status.
2. **Timed Auto-Off**: Add code to turn the relay OFF automatically after 10 seconds of being closed.
3. **Buzzer confirmation tone**: Sound a quick beep on a buzzer (GPIO 15) when the relay state changes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay does not switch | Pin assignment conflict | Verify that the relay signal line is wired to GPIO 13 |
| Commands are ignored | Callback not registered | Ensure `client.setCallback(callback)` is called in setup |
| The ESP32 does not receive commands | `client.loop()` missing | Verify that `client.loop()` is called continuously in the loop function |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [46 - ESP32 Web-based Relay switch control](../beginner/46-esp32-web-based-relay-switch-control.md)
- [79 - ESP32 MQTT Topic subscriber (Remote pin control command)](79-esp32-mqtt-topic-subscriber-remote-pin-control-command.md)
- [80 - ESP32 MQTT Two-Way control broker link](80-esp32-mqtt-two-way-control-broker-link.md)
