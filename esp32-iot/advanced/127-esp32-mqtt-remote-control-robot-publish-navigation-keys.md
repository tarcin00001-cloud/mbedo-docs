# 127 - MQTT Remote Control Robot (Publish navigation keys)

Build a cloud-connected mobile robot on the ESP32 that subscribes to an MQTT command topic on `broker.hivemq.com` to receive navigation payloads, controls two DC motors using an L298N motor driver (IN1–IN4 on GPIOs 12, 14, 27, and 26), and publishes feedback states to a status topic.

## Goal
Learn how to establish MQTT broker handshakes, process incoming command payloads using callback handlers, control L298N motor driver directions, and publish feedback telemetry packets.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and subscribes to the MQTT topic `esp32/robot/command`. An L298N driver on GPIOs 12, 14, 27, and 26 controls two DC motors. Sending control characters (e.g. `F`, `B`, `L`, `R`, `S`) to the command topic via a client (like MQTTX or a smartphone app) drives the robot. The ESP32 publishes status updates (e.g. `DIR:FORWARD`) to the topic `esp32/robot/status` in response.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N H-Bridge Motor Driver | `l298n` | Yes | Yes |
| DC Gear Motors | `motor_dc` | Yes (2 pieces) | Yes (2 pieces) |
| Shared power / Battery Pack | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, monitor the motor widgets to verify that they rotate in response to MQTT commands.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Driver | IN1 / IN2 | GPIO12 / GPIO14 | Yellow / Green | Left motor control |
| L298N Driver | IN3 / IN4 | GPIO27 / GPIO26 | Blue / White | Right motor control |
| L298N Driver | GND | GND | Black | Common ground rail |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the L298N driver and motors from a separate battery pack and connect the battery ground to the ESP32 ground pin.

## Code
```cpp
// MQTT Remote Control Robot (MQTT Callback Command processing + Feedback publisher)
#include <WiFi.h>
#include <PubSubClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Public MQTT Broker configuration
const char* mqttServer = "broker.hivemq.com";
const int mqttPort = 1883;

// Unique client identifier
const char* mqttClientId = "esp32_robot_ctrl_99";
const char* topicCmd = "esp32/robot/command";
const char* topicStatus = "esp32/robot/status";

// Left Motor Control Pins
const int IN1 = 12;
const int IN2 = 14;

// Right Motor Control Pins
const int IN3 = 27;
const int IN4 = 26;

WiFiClient espClient;
PubSubClient client(espClient);

String currentDirection = "STOP";
unsigned long lastReconnectAttempt = 0;

// Movement control routines
void moveForward() {
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

// Publish robot state feedback payload
void publishRobotStatus() {
  String payload = "{\"direction\":\"" + currentDirection + "\"}";
  client.publish(topicStatus, payload.c_str());
  Serial.printf("[MQTT] Status published -> %s\n", payload.c_str());
}

// MQTT Message received callback
void mqttCallback(char* topic, byte* payload, unsigned int length) {
  String message = "";
  for (unsigned int i = 0; i < length; i++) {
    message += (char)payload[i];
  }
  message.trim();
  
  Serial.printf("[MQTT Message] Received on topic [%s]: %s\n", topic, message.c_str());
  
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
  
  publishRobotStatus();
}

// Non-blocking broker reconnect handler
bool reconnectMQTT() {
  if (client.connect(mqttClientId)) {
    Serial.println("[MQTT] Connected to broker.");
    
    // Subscribe to commands topic
    client.subscribe(topicCmd);
    Serial.printf("[MQTT] Subscribed to topic: %s\n", topicCmd);
    
    // Publish armed status
    client.publish(topicStatus, "{\"status\":\"ARMED\"}");
    return true;
  }
  Serial.print("[MQTT] Handshake failed, state: ");
  Serial.println(client.state());
  return false;
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Configure motor driver pins
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  stopRobot();
  
  Serial.println("\nESP32 MQTT Robot Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Setup MQTT server and callback
  client.setServer(mqttServer, mqttPort);
  client.setCallback(mqttCallback);
  
  lastReconnectAttempt = 0;
}

void loop() {
  // Enforce non-blocking MQTT broker client loop connection
  if (!client.connected()) {
    unsigned long now = millis();
    if (now - lastReconnectAttempt >= 5000) { // Retry connection every 5 seconds
      lastReconnectAttempt = now;
      if (reconnectMQTT()) {
        lastReconnectAttempt = 0;
      }
    }
  } else {
    client.loop(); // Process broker keep-alives and incoming payloads
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N Driver**, and **2 DC Motors** onto the canvas.
2. Wire IN1/IN2 to **GPIO12/GPIO14**, and IN3/IN4 to **GPIO27/GPIO26**. Wire the motor outputs of the L298N to the motors.
3. Paste the code and click **Run**.
4. Open an MQTT client (such as MQTTX or a web dashboard tool) and connect to the broker address `broker.hivemq.com`.
5. Publish the character `F` to topic `esp32/robot/command`. Verify that the simulated motor widgets on the canvas spin forward.
6. Publish `S` to stop them. Verify that the ESP32 publishes state feedback to topic `esp32/robot/status`.

## Expected Output
Serial Monitor:
```
WiFi Connected.
[MQTT] Connected to broker.
[MQTT] Subscribed to topic: esp32/robot/command
[MQTT Message] Received on topic [esp32/robot/command]: F
[MQTT] Status published -> {"direction":"FORWARD"}
```

## Expected Canvas Behavior
* Publishing control characters via MQTT spins or stops the simulated motor widgets on the canvas instantly.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `client.setCallback(mqttCallback)` | Binds the incoming message callback handler. |
| `client.subscribe(topicCmd)` | Subscribes to the commands topic on connection. |
| `message == "FORWARD"` | Classifies the received payload string. |
| `client.publish(topicStatus, ...)` | Transmits robot status payload back to the broker. |

## Hardware & Safety Concept: Fail-Safe Timeout and Keep-Alives
Unlike WebSockets, which detect a disconnect almost instantly, an MQTT broker can take up to 15 seconds to detect a connection drop (based on the Keep-Alive interval). If a robot is moving forward when connection is lost, it will keep moving for 15 seconds before the broker registers the drop. To prevent collisions:
1. **Local Heartbeat Timer**: The robot code should monitor incoming commands. If no command is received for more than 2 seconds, the code should call `stopRobot()` automatically as a fail-safe.
2. **LWT (Last Will and Testament)**: Configure the MQTT connection with a Last Will message on the status topic (e.g. `{"status":"OFFLINE"}`) to notify other clients if the robot drops offline.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a lock icon showing the connection status.
2. **Reverse buzzer**: Add a buzzer (GPIO 15) to sound a warning beep when the reverse command (`B`) is active.
3. **SPIFFS integration**: Log received command history to a file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| MQTT connection fails | Broker offline or blocked | Public brokers can experience high loads. Try using a different public broker like `test.mosquitto.org` |
| Motor direction reversed | Wiring reversed | Swap the output wires on the motor driver for that motor |
| Commands are ignored | Client ID collision | Public brokers require unique client IDs. Change `esp32_robot_ctrl_99` to a custom string |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [80 - ESP32 MQTT Two-Way control broker link](80-esp32-mqtt-two-way-control-broker-link.md)
- [121 - ESP32 WiFi Web Page Drive Controller](121-esp32-wifi-web-page-drive-controller.md)
- [128 - ESP32 MQTT Robot with Crash Override](128-esp32-mqtt-robot-with-crash-override-logs-published-to-cloud.md) (Next project)
