# 83 - MQTT Subscribe Servo Position controller

Configure the ESP32 as an MQTT subscriber node, register a callback handler, subscribe to the topic `esp32/control/servo`, drive a servo motor connected to GPIO 13 to target angles (0° to 180°) received in payload strings, and publish confirmation feedback to `esp32/status/servo`.

## Goal
Learn how to subscribe to MQTT topics, parse integer values from payloads, position servo motors using `ESP32Servo`, and publish confirmation feedback.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A servo motor is on GPIO 13. The ESP32 connects to the HiveMQ broker and subscribes to `esp32/control/servo`. Publishing any integer between 0 and 180 positions the servo motor shaft to that angle. The ESP32 publishes a confirmation payload (e.g. `ANGLE:90`) to `esp32/status/servo`.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Servo Motor | PWM (Signal) | GPIO13 | Orange | Servo signal line |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the servo from the 5V Vin rail.

## Code
```cpp
// MQTT Subscribe Servo Position controller (Actuator positioning receiver node)
#include <WiFi.h>
#include <PubSubClient.h>
#include <ESP32Servo.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const char* mqttServer = "broker.hivemq.com";
const int mqttPort = 1883;

// Configure MQTT Topics
const char* commandTopic = "esp32/control/servo";
const char* feedbackTopic = "esp32/status/servo";

const int SERVO_PIN = 13;
Servo myServo;

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
  
  Serial.printf("[MQTT Sub] Positioning Command arrived -> Topic: %s | Payload: %s\n", topic, message.c_str());
  
  // Verify command topic
  if (String(topic) == commandTopic) {
    int angle = message.toInt();
    
    // Validate target angle within bounds
    if (angle >= 0 && angle <= 180) {
      // Write target angle to servo motor
      myServo.write(angle);
      Serial.printf("[Action] Servo positioned to %d degrees.\n", angle);
      
      // Publish confirmation feedback
      String feedbackPayload = "ANGLE:" + String(angle);
      client.publish(feedbackTopic, feedbackPayload.c_str());
    } else {
      Serial.printf("[Error] Out of bounds angle attempted: %d degrees\n", angle);
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
      client.publish(feedbackTopic, "ANGLE:90");
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
  
  // Configure and attach Servo motor
  myServo.attach(SERVO_PIN);
  myServo.write(90); // Center servo (90 degrees) on startup
  
  Serial.println("\nESP32 MQTT Servo Subscriber Starting...");
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
1. Drag **ESP32 DevKitC** and **Servo** onto the canvas.
2. Wire Servo to **GPIO13**.
3. Paste the code and click **Run**.
4. In simulation, publish an integer value (e.g. `45` or `135`) to the topic `esp32/control/servo` using an MQTT client utility. Watch the servo rotate.

## Expected Output
Serial Monitor:
```
ESP32 MQTT Servo Subscriber Starting...
....
WiFi Connected.
Attempting MQTT connection...connected.
Subscribed to topic: esp32/control/servo
[MQTT Sub] Positioning Command arrived -> Topic: ... | Payload: 45
[Action] Servo positioned to 45 degrees.
[MQTT Sub] Positioning Command arrived -> Topic: ... | Payload: 180
[Action] Servo positioned to 180 degrees.
```

## Expected Canvas Behavior
* Publishing an integer to the topic rotates the servo widget in real time.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `myServo.attach(SERVO_PIN)` | Configures the ESP32 hardware timer to drive the servo signal pin. |
| `myServo.write(angle)` | Sends the target angle command to the servo controller. |
| `client.publish(feedbackTopic, ...)` | Publishes a status confirmation payload back to the broker. |
| `client.loop()` | Processes background frames to parse subscription events. |

## Hardware & Safety Concept: Actuator Speed Limits and Current Spikes
Servo motors draw significant current when moving (up to 500 mA). If powered directly from the ESP32's 3.3V rail, they can drop the voltage and reset the microcontroller. Always power the servo from the **5V Vin rail** or an external power supply. Additionally, validate that the angle is restricted between 0° and 180° to prevent the servo from hitting its physical limits and drawing excessive current.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the current servo angle.
2. **Angle sweep command**: Add a command `SWEEP` to sweep the servo back and forth automatically between 0° and 180°.
3. **Buzzer limit alert**: Sound a quick beep on a buzzer (GPIO 15) when the servo reaches 0° or 180° limits.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo does not move | Vin power missing | Confirm that the servo is powered from the 5V Vin rail, not the 3.3V rail |
| Commands are ignored | Callback not registered | Ensure `client.setCallback(callback)` is called in setup |
| Servo jitter | Noise on signal line | Install a 100 µF capacitor across the servo power lines to filter noise |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [56 - ESP32 Web Page Servo angle controller](56-esp32-web-page-servo-angle-controller-oled-display-feedback.md)
- [81 - ESP32 MQTT Subscribe Relay controller](81-esp32-mqtt-subscribe-relay-controller.md)
- [82 - ESP32 MQTT Subscribe Buzzer melody player](82-esp32-mqtt-subscribe-buzzer-melody-player.md)
