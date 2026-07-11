# 84 - MQTT Auto-Reconnect state loops

Configure a robust, non-blocking MQTT reconnection state machine on the ESP32 that uses `millis()` timers to schedule reconnection attempts without halting CPU execution, and blinks a status LED on GPIO 13 continuously to demonstrate active local processing during broker outages.

## Goal
Learn how to implement non-blocking reconnection routines, compare blocking vs non-blocking state loops, handle reconnect timers, and design resilient firmware.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and attempts to join an MQTT broker. If the broker is offline, the ESP32 logs the failure and continues to blink an LED on GPIO 13 in the background. It schedules reconnect attempts every 5 seconds without blocking, ensuring local control loops remain active.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Red LED | Anode (+) | GPIO13 via 330 Ω | Red | Heartbeat status indicator |
| Shared ground | GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the Red LED anode to GPIO 13 via a 330 Ω resistor.

## Code
```cpp
// MQTT Auto-Reconnect state loops (Non-blocking MQTT Reconnection Machine)
#include <WiFi.h>
#include <PubSubClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Configure MQTT Broker details
const char* mqttServer = "broker.hivemq.com";
const int mqttPort = 1883;

const int LED_PIN = 13;

WiFiClient espClient;
PubSubClient client(espClient);

// Reconnection State Variables
unsigned long lastReconnectAttempt = 0;
const unsigned long RECONNECT_INTERVAL_MS = 5000; // Retry every 5 seconds

// Heartbeat Blinking Variables
unsigned long lastBlinkTime = 0;
const unsigned long BLINK_INTERVAL_MS = 500; // Blink LED every 500 ms
bool ledState = LOW;

void callback(char* topic, byte* payload, unsigned int length) {
  // Callback details
}

// 1. Non-Blocking MQTT Reconnection routine
// Attempts connection and returns immediately without halting execution
bool attemptMQTTConnect() {
  Serial.print("[MQTT] Initiating non-blocking connection attempt...");
  
  // Generate unique Client ID
  String clientId = "ESP32ClientResilient-" + String(random(0xffff), HEX);
  
  if (client.connect(clientId.c_str())) {
    Serial.println("Success! Connected to broker.");
    client.publish("esp32/resilience/status", "ONLINE");
    return true;
  } else {
    Serial.print("Failed! state code = ");
    Serial.println(client.state());
    return false;
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  
  Serial.println("\nESP32 Resilient Client Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
  
  client.setServer(mqttServer, mqttPort);
  client.setCallback(callback);
}

void loop() {
  unsigned long now = millis();
  
  // 2. Continuous Heartbeat LED Blink (Demonstrates local CPU activity)
  if (now - lastBlinkTime >= BLINK_INTERVAL_MS) {
    ledState = !ledState;
    digitalWrite(LED_PIN, ledState);
    lastBlinkTime = now;
  }
  
  // 3. Non-Blocking MQTT Reconnection manager
  if (!client.connected()) {
    // Check if the reconnect interval has elapsed
    if (now - lastReconnectAttempt >= RECONNECT_INTERVAL_MS) {
      lastReconnectAttempt = now;
      
      // Attempt connection. If it fails, the code exits immediately
      // and continues the loop, allowing the LED to keep blinking
      attemptMQTTConnect();
    }
  } else {
    // Process active subscription events if connected
    client.loop();
  }
  
  // Minor delay to yield CPU to WiFi system tasks
  delay(1);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Wire LED to **GPIO13**.
3. Paste the code and click **Run**.
4. Disconnect the simulated WiFi network (or stop the broker).
5. Watch the Serial Monitor log reconnect attempts every 5 seconds. Verify that the LED widget continues to blink smoothly throughout the process.

## Expected Output
Serial Monitor:
```
ESP32 Resilient Client Starting...
....
WiFi Connected successfully.
[MQTT] Initiating non-blocking connection attempt...Success! Connected to broker.
```

If connection drops:
```
[MQTT] Initiating non-blocking connection attempt...Failed! state code = -2
[MQTT] Initiating non-blocking connection attempt...Failed! state code = -2
```

## Expected Canvas Behavior
* The Red LED widget blinks at a steady rate of 2 Hz (every 500 ms) regardless of whether the MQTT broker is connected or disconnected.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `now - lastReconnectAttempt >= 5000` | Checks if the 5-second interval has elapsed before initiating a reconnect attempt. |
| `attemptMQTTConnect()` | Initiates the connection, returning immediately without blocking delays. |
| `lastBlinkTime` | Tracks the blink timer to toggle the LED asynchronously. |

## Hardware & Safety Concept: Blocking vs Non-Blocking Code in Safety Systems
Traditional MQTT reconnection code uses a blocking `while` loop that pauses execution until the broker responds. If the network goes offline, the device freezes for several seconds. In safety-critical systems (such as battery heat monitors or limit switches), freezing the CPU can prevent reading sensor bounds, causing failures. Enforcing **non-blocking timers** ensures local safety checks remain active.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a count of failed connection attempts.
2. **Dynamic Backoff Interval**: Increase the reconnection interval (e.g. from 5s to 10s to 30s) after consecutive failures to reduce network traffic.
3. **Buzzer alert on join**: Sound a quick beep on a buzzer (GPIO 15) when the connection completes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The LED stops blinking during reconnect | Blocking delay in reconnect | Verify that the reconnection loop does not contain `delay()` statements |
| Connection fails with rc = -2 | Broker unreachable / DNS failed | Verify the broker hostname is correct and the gateway has internet access |
| Client ID conflict | Client ID not unique | Ensure you generate a unique Client ID for each connection |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [76 - ESP32 MQTT Client Connection](76-esp32-mqtt-client-connection.md)
- [80 - ESP32 MQTT Two-Way control broker link](80-esp32-mqtt-two-way-control-broker-link.md)
- [85 - ESP32 MQTT JSON Payload publisher](85-esp32-mqtt-json-payload-publisher-serialize-dht22-values.md) (Next project)
