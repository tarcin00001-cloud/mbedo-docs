# 128 - MQTT Robot with Crash Override logs published to cloud

Build an MQTT-controlled mobile robot on the ESP32 that features an autonomous crash override using an HC-SR04 ultrasonic range-finder (Trig on GPIO 12, Echo on GPIO 14), drives motors using an L298N driver, and publishes critical collision warning logs to the cloud.

## Goal
Learn how to implement local sensor safety overrides within network-controlled loops, combine ultrasonic range finding with motor driver controls, build MQTT clients, and publish JSON alert payloads.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network, subscribing to MQTT commands (`esp32/robot/command`) to drive. An HC-SR04 ultrasonic sensor is on GPIO 12/14. If the robot receives a command to drive forward, but the ultrasonic sensor detects an obstacle closer than 20 cm, the ESP32 activates a **Crash Override**: it stops the motors, ignores the forward command, and publishes a warning log to the topic `esp32/robot/alerts`.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| L298N H-Bridge Motor Driver | `l298n` | Yes | Yes |
| DC Gear Motors | `motor_dc` | Yes (2 pieces) | Yes (2 pieces) |
| Shared power / Battery Pack | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 Sensor | TRIG | GPIO12 | Yellow | Trigger pulse output |
| HC-SR04 Sensor | ECHO | GPIO14 | Green | Echo pulse input |
| L298N Driver | IN1 / IN2 | GPIO27 / GPIO26 | Blue / White | Left motor control |
| L298N Driver | IN3 / IN4 | GPIO25 / GPIO33 | Purple / Grey | Right motor control |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the HC-SR04 sensor and L298N driver from the 5V Vin rail.

## Code
```cpp
// MQTT Robot with Crash Override (Ultrasonic safety loops + Cloud warning publisher)
#include <WiFi.h>
#include <PubSubClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Public MQTT Broker configuration
const char* mqttServer = "broker.hivemq.com";
const int mqttPort = 1883;

const char* mqttClientId = "esp32_safety_bot_99";
const char* topicCmd = "esp32/robot/command";
const char* topicStatus = "esp32/robot/status";
const char* topicAlerts = "esp32/robot/alerts";

// HC-SR04 Pins
const int TRIG_PIN = 12;
const int ECHO_PIN = 14;

// L298N motor driver direction pins
const int IN1 = 27;
const int IN2 = 26;
const int IN3 = 25;
const int IN4 = 33;

WiFiClient espClient;
PubSubClient client(espClient);

float currentDistanceCm = 100.0;
String currentDirection = "STOP";
bool crashOverrideActive = false;
unsigned long lastReconnectAttempt = 0;

// Read HC-SR04 sensor distance in cm
float readDistanceCm() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  long duration = pulseIn(ECHO_PIN, HIGH, 30000); // 30ms timeout (max 5 meters)
  if (duration == 0) return 400.0;
  
  return (float)duration * 0.0343 / 2.0;
}

// Drive control routines
void moveForward() {
  // Enforce safety override check
  if (currentDistanceCm < 20.0) {
    Serial.println("[Safety Alert] Drive FORWARD blocked by Crash Override!");
    return;
  }
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  currentDirection = "FORWARD";
}

void moveBackward() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  currentDirection = "REVERSE";
}

void turnLeft() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  currentDirection = "LEFT";
}

void turnRight() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  currentDirection = "RIGHT";
}

void stopRobot() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
  currentDirection = "STOP";
}

// Publish state feedback
void publishStatus() {
  String payload = "{\"direction\":\"" + currentDirection + "\",\"distance\":" + String(currentDistanceCm, 1) + "}";
  client.publish(topicStatus, payload.c_str());
}

// Publish crash alert log to cloud
void publishCrashAlert() {
  String payload = "{\"alert\":\"CRASH_OVERRIDE\",\"distance\":" + String(currentDistanceCm, 1) + "}";
  client.publish(topicAlerts, payload.c_str());
  Serial.println("[MQTT ALERT] Collision warning log published to cloud!");
}

// MQTT Message received callback
void mqttCallback(char* topic, byte* payload, unsigned int length) {
  String message = "";
  for (unsigned int i = 0; i < length; i++) {
    message += (char)payload[i];
  }
  message.trim();
  
  Serial.printf("[MQTT Message] Received: %s\n", message.c_str());
  
  // Parse command key codes
  if (message == "F" || message == "FORWARD") {
    moveForward();
  } 
  else if (message == "B" || message == "REVERSE") {
    moveBackward();
  } 
  else if (message == "L" || message == "LEFT") {
    turnLeft();
  } 
  else if (message == "R" || message == "RIGHT") {
    turnRight();
  } 
  else if (message == "S" || message == "STOP") {
    stopRobot();
  }
  
  publishStatus();
}

// Non-blocking broker reconnect handler
bool reconnectMQTT() {
  if (client.connect(mqttClientId)) {
    Serial.println("[MQTT] Connected to broker.");
    client.subscribe(topicCmd);
    client.publish(topicStatus, "{\"status\":\"ARMED\"}");
    return true;
  }
  return false;
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  stopRobot();
  
  Serial.println("\nESP32 Safety Robot Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  client.setServer(mqttServer, mqttPort);
  client.setCallback(mqttCallback);
  
  lastReconnectAttempt = 0;
}

void loop() {
  // Enforce non-blocking connection recovery
  if (!client.connected()) {
    unsigned long now = millis();
    if (now - lastReconnectAttempt >= 5000) {
      lastReconnectAttempt = now;
      if (reconnectMQTT()) {
        lastReconnectAttempt = 0;
      }
    }
  } else {
    client.loop();
  }
  
  // Periodically sample ultrasonic distance (every 100 ms)
  static unsigned long lastDistanceScan = 0;
  if (millis() - lastDistanceScan >= 100) {
    currentDistanceCm = readDistanceCm();
    lastDistanceScan = millis();
    
    // Safety check: override and stop if driving forward and an obstacle appears
    if (currentDirection == "FORWARD" && currentDistanceCm < 20.0) {
      stopRobot();
      crashOverrideActive = true;
      publishCrashAlert();
      publishStatus();
    }
    
    // Clear override once path is clear
    if (crashOverrideActive && currentDistanceCm > 30.0) {
      crashOverrideActive = false;
      Serial.println("[Safety] Path cleared. Crash override reset.");
    }
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **HC-SR04 Sensor**, **L298N Driver**, and **2 DC Motors** onto the canvas.
2. Wire TRIG/ECHO to **GPIO12/GPIO14**, and IN1–IN4 to **GPIO27, 26, 25, and 33**.
3. Paste the code and click **Run**.
4. Set the simulated ultrasonic sensor distance to a high value (e.g. 100 cm).
5. Open your MQTT client, connect to `broker.hivemq.com`, and publish the string `F` to topic `esp32/robot/command`. Verify that the simulated motors spin forward.
6. Decrease the simulated ultrasonic distance slider below 20 cm. Verify that the motor widgets stop immediately, and the ESP32 publishes a crash log to topic `esp32/robot/alerts`.

## Expected Output
Serial Monitor:
```
WiFi Connected.
[MQTT] Connected to broker.
[MQTT Message] Received: F
[MQTT ALERT] Collision warning log published to cloud!
```

## Expected Canvas Behavior
* Triggering a collision state by sliding the simulated distance below 20 cm stops the motor widgets instantly.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `readDistanceCm()` | Samples the distance in front of the robot. |
| `currentDirection == "FORWARD" && currentDistanceCm < 20.0` | Safety trigger condition checking for obstacle collision. |
| `stopRobot()` | Shuts down the motors to prevent crash. |
| `client.publish(topicAlerts, ...)` | Publishes the warning log JSON payload to the alerts topic. |

## Hardware & Safety Concept: Local Safety Loops vs Cloud Latency
Cloud networks introduce latency (typically 50–500 ms). If a robot relies on a remote cloud server to process sensor data and send a stop command, it will likely hit the obstacle before the stop command arrives. Implementing a **local safety loop** directly on the ESP32 ensures the robot stops instantly (under 10 ms), while publishing logs to the cloud afterward for diagnostics.

## Try This! (Challenges)
1. **OLED Warning Display**: Add an OLED screen (Project 60) and display a warning graphic during a crash override.
2. **Alert Buzzer**: Add a buzzer (GPIO 15) to sound an alarm when a crash override is active.
3. **Recovery Routine**: implement an automatic backup-and-turn recovery routine when a collision is detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot does not respond to MQTT commands | Client ID collision | Public brokers require unique client IDs. Change `esp32_safety_bot_99` to a custom string |
| The override does not trigger | Scan interval too slow | Verify that the distance is sampled frequently (e.g. every 100 ms) |
| Motors spin backwards | Wiring reversed | Swap the output wires on the motor driver for that motor |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [124 - ESP32 Obstacle Avoidance Robot with Web HUD](124-esp32-obstacle-avoidance-robot-with-web-hud-telemetry.md)
- [127 - ESP32 MQTT Remote Control Robot](127-esp32-mqtt-remote-control-robot-publish-navigation-keys.md)
- [129 - Robot Wall Follower Web Graph display](129-robot-wall-follower-web-graph-display.md) (Next project)
