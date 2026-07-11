# 198 - Bluetooth & WiFi Robot HUD Console (HC-05 + HC-SR04 + Web sockets)

Build a dual-radio telemetry gateway robot on the ESP32 that receives drive navigation commands over Classic Bluetooth Serial (HC-05 module), controls DC motors, samples an HC-SR04 distance sensor, and broadcasts live telemetry to a WebSocket HUD dashboard.

## Goal
Learn how to configure hardware serial UART ports on the ESP32, interface HC-05 Bluetooth modules, execute dual-protocol bridging loops (Bluetooth-to-WiFi), stream live WebSocket data, and build web HUD diagnostics.

## What You Will Build
An ESP32 DevKitC acts as a robot vehicle controller. It is wired to an HC-05 Bluetooth module (Serial2 on GPIO 16/17), an L298N motor driver, and an HC-SR04 ultrasonic sensor (GPIO 14, 15). The user pairs their phone with the HC-05 module and sends drive characters ('F', 'B', 'L', 'R', 'S'). Simultaneously, the ESP32 connects to WiFi and hosts a WebSocket dashboard. Computers on the network can view a visual HUD plotting the current drive state and distance readings in real-time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| L298N Motor Driver + 2x DC Motors | `l298n` / `motor` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-05 | TXD | GPIO16 | Yellow | Hardware Serial2 RX |
| HC-05 | RXD | GPIO17 | Green | Hardware Serial2 TX (uses voltage divider) |
| HC-SR04 | Trigger | GPIO14 | Blue | Trigger pulse input |
| HC-SR04 | Echo | GPIO15 | Violet | Echo pulse output |
| L298N Driver | IN1 / IN2 | GPIO25 / GPIO26 | Orange / Yellow | Left Motor speed control |
| L298N Driver | IN3 / IN4 | GPIO32 / GPIO33 | Grey / White | Right Motor speed control |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Solder a 1 kΩ and 2 kΩ resistor voltage divider on the HC-05 RXD pin to step down the ESP32's 3.3V TX signal to the HC-05's 5V logic level.

## Code
```cpp
// Bluetooth-WiFi Bridge Robot (Hardware Serial2 UART + HC-SR04 + L298N PWM + WebSockets Broadcast)
#include <WiFi.h>
#include <WebServer.h>
#include <WebSocketsServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// HC-05 Bluetooth Serial (Hardware Serial 2)
// Default pins: RX2 = GPIO 16, TX2 = GPIO 17
#define RXD2 16
#define TXD2 17

// HC-SR04 Pins
const int TRIG_PIN = 14;
const int ECHO_PIN = 15;

// Motor Driver Pins
const int IN1 = 25;
const int IN2 = 26;
const int IN3 = 32;
const int IN4 = 33;

WebServer server(80);
WebSocketsServer webSocket(81);

float obstacleDistanceCm = 100.0;
String activeCommand = "RELEASE (STOP)";

// Drive motor functions
void moveForward() {
  analogWrite(IN1, 180); digitalWrite(IN2, LOW);
  analogWrite(IN3, 180); digitalWrite(IN4, LOW);
}

void moveBackward() {
  digitalWrite(IN1, LOW); analogWrite(IN2, 180);
  digitalWrite(IN3, LOW); analogWrite(IN4, 180);
}

void turnLeft() {
  digitalWrite(IN1, LOW); analogWrite(IN2, 180);
  analogWrite(IN3, 180); digitalWrite(IN4, LOW);
}

void turnRight() {
  analogWrite(IN1, 180); digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW); analogWrite(IN4, 180);
}

void stopRobot() {
  digitalWrite(IN1, LOW); digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW); digitalWrite(IN4, LOW);
}

// Process single character command strings
void parseDriveCommand(char cmd) {
  switch (cmd) {
    case 'F':
      moveForward();
      activeCommand = "DRIVE FORWARD";
      break;
    case 'B':
      moveBackward();
      activeCommand = "DRIVE BACKWARD";
      break;
    case 'L':
      turnLeft();
      activeCommand = "STEER LEFT";
      break;
    case 'R':
      turnRight();
      activeCommand = "STEER RIGHT";
      break;
    case 'S':
      stopRobot();
      activeCommand = "RELEASE (STOP)";
      break;
    default:
      break;
  }
  Serial.printf("[Bluetooth Command] Command: %c (%s)\n", cmd, activeCommand.c_str());
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Robot Dual HUD</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 540px; width: 100%; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; text-align: center; margin-top: 0; }\n";
  html += "  .hud-display { text-align: center; padding: 20px; border-radius: 8px; font-family: monospace; font-size: 20px; font-weight: bold; background-color: #0f172a; border: 1px solid #334155; margin: 20px 0; color: #38bdf8; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 20px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 24px; font-weight: bold; margin-top: 5px; color: #10b981; }\n";
  html += "  canvas { width: 100%; height: 180px; background-color: #0f172a; border-radius: 8px; border: 1px solid #334155; }\n";
  html += "  .footer { text-align: center; margin-top: 25px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Robotic Dual Radio HUD</h1>\n";
  
  html += "  <div class=\"hud-display\" id=\"hudCommand\">RELEASE (STOP)</div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Ultrasonic Range</div><div class=\"metric-val\" id=\"hudDist\">100.0 cm</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Radio Active</div><div class=\"metric-val\" style='color:#a7f3d0;'>BT + WiFi</div></div>\n";
  html += "  </div>\n";
  
  html += "  <canvas id=\"chart\"></canvas>\n";
  
  html += "  <p class=\"footer\">BT: HC-05 (Serial2 9600) | WiFi WebSocket: Port 81</p>\n";
  html += "</div>\n";
  
  // Real-time Canvas Graphing Script
  html += "<script>\n";
  html += "  const canvas = document.getElementById('chart');\n";
  html += "  const ctx = canvas.getContext('2d');\n";
  html += "  const history = new Array(80).fill(100);\n";
  
  html += "  function drawChart() {\n";
  html += "    ctx.clearRect(0, 0, canvas.width, canvas.height);\n";
  html += "    ctx.beginPath();\n";
  html += "    ctx.strokeStyle = '#10b981';\n";
  html += "    ctx.lineWidth = 2.5;\n";
  
  html += "    const step = canvas.width / (history.length - 1);\n";
  html += "    for (let i = 0; i < history.length; i++) {\n";
  html += "      const y = canvas.height - (history[i] / 200.0) * canvas.height;\n";
  html += "      if (i === 0) ctx.moveTo(0, y); else ctx.lineTo(i * step, y);\n";
  html += "    }\n";
  html += "    ctx.stroke();\n";
  html += "  }\n";
  
  // WebSockets handler
  html += "  const ws = new WebSocket('ws://' + window.location.hostname + ':81/');\n";
  html += "  ws.onmessage = function(event) {\n";
  html += "    const data = JSON.parse(event.data);\n";
  html += "    document.getElementById('hudCommand').innerText = data.cmd;\n";
  html += "    document.getElementById('hudDist').innerText = data.dist.toFixed(1) + ' cm';\n";
  
  html += "    history.push(data.dist);\n";
  html += "    history.shift();\n";
  html += "    drawChart();\n";
  html += "  };\n";
  
  html += "  window.onload = function() {\n";
  html += "    canvas.width = canvas.clientWidth;\n";
  html += "    canvas.height = canvas.clientHeight;\n";
  html += "    drawChart();\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // 1. Initialize Hardware Serial 2 (Baud Rate 9600 for HC-05)
  Serial2.begin(9600, SERIAL_8N1, RXD2, TXD2);
  Serial.println("[Bluetooth] Serial2 UART started (9600 Baud).");
  
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  stopRobot();
  
  Serial.println("\nESP32 Dual-Radio Robot starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  Serial.print("Local IP Address: ");
  Serial.println(WiFi.localIP());
  
  // Start servers
  server.on("/", HTTP_GET, handleRoot);
  server.begin();
  
  webSocket.begin();
  Serial.println("HTTP and WebSocket servers active.");
}

void loop() {
  server.handleClient();
  webSocket.loop();
  
  // 2. Poll incoming Bluetooth serial buffer
  if (Serial2.available() > 0) {
    char cmd = Serial2.read();
    parseDriveCommand(cmd);
  }
  
  // Mock simulation commands from serial terminal
  if (Serial.available() > 0) {
    char cmd = Serial.read();
    if (cmd == 'F' || cmd == 'B' || cmd == 'L' || cmd == 'R' || cmd == 'S') {
      parseDriveCommand(cmd);
    }
  }
  
  // 3. Measure ultrasonic obstacle range
  static unsigned long lastDistanceCheck = 0;
  if (millis() - lastDistanceCheck > 100) {
    lastDistanceCheck = millis();
    
    digitalWrite(TRIG_PIN, LOW);
    delayMicroseconds(2);
    digitalWrite(TRIG_PIN, HIGH);
    delayMicroseconds(10);
    digitalWrite(TRIG_PIN, LOW);
    
    long duration = pulseIn(ECHO_PIN, HIGH, 30000);
    if (duration > 0) {
      obstacleDistanceCm = duration * 0.0343 / 2.0;
    }
    
    // Auto-stop safety override: if obstacle is closer than 15 cm
    if (obstacleDistanceCm < 15.0 && activeCommand != "RELEASE (STOP)") {
      stopRobot();
      activeCommand = "CRASH OVERRIDE (AUTO-STOP)";
      Serial.println("[Safety Override] Obstacle too close! Robot stopped.");
    }
  }
  
  // 4. Broadcast system state to WebSockets (every 200 ms)
  static unsigned long lastBroadcast = 0;
  if (millis() - lastBroadcast > 200) {
    lastBroadcast = millis();
    
    String jsonPayload = "{\"cmd\":\"" + activeCommand + "\",\"dist\":" + String(obstacleDistanceCm, 1) + "}";
    webSocket.broadcastTXT(jsonPayload);
  }
  
  delay(1);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **HC-05 Bluetooth**, **HC-SR04**, and **L298N Driver** onto the canvas.
2. Wire the Trigger and Echo pins to **GPIO14** and **GPIO15**. Wire Serial2 RX/TX to **GPIO16** and **GPIO17**.
3. Paste the code and click **Run**.
4. In simulation, mock Bluetooth messages by typing driving letters (e.g. `F` for forward, `L` for left) in the serial terminal.
5. Verify that the motor driver DC widgets spin in correct directions.
6. Slide the simulated HC-SR04 range down to 10 cm. Verify that the motors turn OFF instantly, overriding the drive state.
7. Open your web browser, navigate to the IP address, and verify that the dashboard graphs the distance and displays the active steering command.

## Expected Output
Serial Monitor:
```
[Bluetooth] Serial2 UART started (9600 Baud).
WiFi Connected.
Local IP Address: 10.10.0.3
HTTP and WebSocket servers active.
[Bluetooth Command] Command: F (DRIVE FORWARD)
[Safety Override] Obstacle too close! Robot stopped.
```

## Expected Canvas Behavior
* Sending simulated serial characters steers the motor widgets on the canvas. Toggling the distance slider below 15 cm forces motor shutdowns and generates WebSocket updates.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Serial2.begin(9600, ...)` | Initializes the secondary hardware serial port matching standard HC-05 default baud rates. |
| `Serial2.available()` | Checks if the hardware buffer contains unread Bluetooth packets. |
| `webSocket.broadcastTXT(...)` | Streams updated robot state variables over the port 81 channel. |

## Hardware & Safety Concept: Logic Level Dividers and Voltage Regulators
* **Logic Level Dividers**: While the ESP32 runs strictly on 3.3V logic, the HC-05 Bluetooth module runs on 5V logic. Connecting the HC-05 TXD to the ESP32 RXD is safe, but connecting the ESP32 TXD (3.3V) directly to the HC-05 RXD (5V) can result in unstable communication or module damage. Always include a voltage divider (using 1 kΩ and 2 kΩ resistors) to bring the 5V signal down to 3.3V.
* **Voltage Regulators**: Servos and motor drivers pull substantial current. If your power supply drops below 4.5V, the ESP32's onboard LDO voltage regulator will fail to maintain 3.3V, causing CPU brownout resets. Use a dedicated Buck regulator (like an LM2596) to regulate input voltage down to a stable 5V for the system.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the current active connection state.
2. **Reverse buzzer**: Add a buzzer (GPIO 15) that sounds a pulsing beep sequence when the robot drives in reverse (`B`).
3. **ThingSpeak Log**: Upload collision alerts and crash log timestamps to a remote ThingSpeak channel (Project 140).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Bluetooth commands ignore input | TX/RX pins crossed | Ensure that the HC-05 TX pin connects to the ESP32 RX2 pin (GPIO 16) and the HC-05 RX pin connects to the ESP32 TX2 pin (GPIO 17). Do not connect TX-to-TX or RX-to-RX |
| WebSocket connection drops | Port conflict | Verify that the WebSocket server binds to a separate port (e.g. 81) from the HTTP server (port 80) and that firewalls do not block UDP port 81 |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [102 - Bluetooth Home Controller](../intermediate/102-bt-home-controller.md)
- [125 - Full BT Robot HUD](../expert/125-full-bt-robot-hud-hud-serial-distance-avoidance-lcd.md)
- [199 - 4x4 Password Lock with Attempt Lockout, Local Siren, & IoT Email Alert](199-4x4-password-lock-with-attempt-lockout-local-siren-iot-email-alert.md) (Next project)
