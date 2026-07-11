# 121 - WiFi Web Page Drive Controller (Left, Right, Forward, Back)

Build a web-controlled differential drive robot on the ESP32 that controls two DC motors using an L298N motor driver (IN1–IN4 on GPIOs 12, 14, 27, and 26) and hosts a styled web control interface with button grid layouts and keyboard arrow key listeners.

## Goal
Learn how to interface L298N motor drivers to drive DC motors, implement differential steering navigation, build responsive grid web pages, and capture keyboard events to trigger HTTP POST control requests.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An L298N motor driver controls two DC motors. The driver control lines are connected to GPIOs 12, 14, 27, and 26. Navigating to the ESP32's IP address displays a dashboard. It features control buttons (Forward, Left, Stop, Right, Reverse). Clicking the buttons or pressing the WASD/Arrow keys on your keyboard sends control requests to drive the robot.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N H-Bridge Motor Driver | `l298n` | Yes | Yes |
| DC Gear Motors | `motor_dc` | Yes (2 pieces) | Yes (2 pieces) |
| Shared power / Battery Pack | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use two DC motor widgets to visualize rotation direction and speed changes.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Driver | IN1 / IN2 | GPIO12 / GPIO14 | Yellow / Green | Left motor control |
| L298N Driver | IN3 / IN4 | GPIO27 / GPIO26 | Blue / White | Right motor control |
| L298N Driver | GND | GND | Black | Common ground rail |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the L298N driver and DC motors from a separate battery pack (e.g. 7.4V or 9V) and connect the battery ground to the ESP32 ground pin.

## Code
```cpp
// WiFi Web Page Drive Controller (Differential DC motor drive + Keyboard controls)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Left Motor Control Pins
const int IN1 = 12;
const int IN2 = 14;

// Right Motor Control Pins
const int IN3 = 27;
const int IN4 = 26;

WebServer server(80);
String currentDirection = "STOP";

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

// HTTP API endpoint returning JSON data
void handleGetDriveStatus() {
  String json = "{\"direction\":\"" + currentDirection + "\"}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to update drive state
void handlePostDrive() {
  if (server.hasArg("cmd")) {
    String cmd = server.arg("cmd");
    
    if (cmd == "forward")  moveForward();
    else if (cmd == "reverse") moveBackward();
    else if (cmd == "left")    turnLeft();
    else if (cmd == "right")   turnRight();
    else                       stopRobot();
    
    Serial.printf("[Drive] Action executed -> %s\n", currentDirection.c_str());
    server.send(200, "text/plain", "OK");
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Robot Web Controller</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 400px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .status-badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 14px; text-transform: uppercase; margin: 15px 0; background-color: #334155; color: #94a3b8; }\n";
  html += "  .status-badge.moving { background-color: #065f46; color: #a7f3d0; }\n";
  
  // D-Pad buttons grid layout
  html += "  .control-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 10px; margin: 25px auto; max-width: 260px; }\n";
  html += "  .btn { padding: 18px; font-size: 18px; font-weight: bold; color: white; background-color: #334155; border: 1px solid #475569; border-radius: 12px; cursor: pointer; transition: background-color 0.1s, transform 0.1s; }\n";
  html += "  .btn:active { background-color: #38bdf8; color: #0f172a; transform: scale(0.95); }\n";
  html += "  .btn-stop { background-color: #991b1b; border-color: #b91c1c; }\n";
  html += "  .btn-stop:hover { background-color: #b91c1c; }\n";
  html += "  .empty { visibility: hidden; }\n";
  html += "  .keyboard-tips { font-size: 11px; color: #64748b; margin-top: 20px; line-height: 1.4; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Robot Web Console</h1>\n";
  html += "  <div id=\"dirBadge\" class=\"status-badge\">STOPPED</div>\n";
  
  // D-Pad Grid
  html += "  <div class=\"control-grid\">\n";
  html += "    <div class=\"empty\"></div>\n";
  html += "    <button class=\"btn\" onmousedown=\"sendCmd('forward')\" onmouseup=\"sendCmd('stop')\">&#9650;</button>\n";
  html += "    <div class=\"empty\"></div>\n";
  
  html += "    <button class=\"btn\" onmousedown=\"sendCmd('left')\" onmouseup=\"sendCmd('stop')\">&#9664;</button>\n";
  html += "    <button class=\"btn btn-stop\" onclick=\"sendCmd('stop')\">STOP</button>\n";
  html += "    <button class=\"btn\" onmousedown=\"sendCmd('right')\" onmouseup=\"sendCmd('stop')\">&#9654;</button>\n";
  
  html += "    <div class=\"empty\"></div>\n";
  html += "    <button class=\"btn\" onmousedown=\"sendCmd('reverse')\" onmouseup=\"sendCmd('stop')\">&#9660;</button>\n";
  html += "    <div class=\"empty\"></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"keyboard-tips\">Use **WASD** or **Arrow Keys** on keyboard.<br>Click and hold button widgets on mobile.</p>\n";
  html += "</div>\n";
  
  // Client-side control and keyboard listener
  html += "<script>\n";
  html += "  let lastSentCmd = 'stop';\n";
  
  html += "  function sendCmd(cmdVal) {\n";
  html += "    if (cmdVal === lastSentCmd) return;\n"; // Prevent spamming
  html += "    lastSentCmd = cmdVal;\n";
  
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('cmd', cmdVal);\n";
  html += "    fetch('/api/drive', { method: 'POST', body: body })\n";
  html += "      .then(() => {\n";
  html += "        const bdg = document.getElementById('dirBadge');\n";
  html += "        bdg.innerText = cmdVal.toUpperCase();\n";
  html += "        if (cmdVal === 'stop') {\n";
  html += "          bdg.className = 'status-badge';\n";
  html += "        } else {\n";
  html += "          bdg.className = 'status-badge moving';\n";
  html += "        }\n";
  html += "      });\n";
  html += "  }\n";
  
  // Keyboard listeners
  html += "  document.addEventListener('keydown', function(event) {\n";
  html += "    if (event.repeat) return;\n";
  html += "    switch(event.key) {\n";
  html += "      case 'ArrowUp':    case 'w': sendCmd('forward'); break;\n";
  html += "      case 'ArrowDown':  case 's': sendCmd('reverse'); break;\n";
  html += "      case 'ArrowLeft':  case 'a': sendCmd('left'); break;\n";
  html += "      case 'ArrowRight': case 'd': sendCmd('right'); break;\n";
  html += "    }\n";
  html += "  });\n";
  
  html += "  document.addEventListener('keyup', function(event) {\n";
  html += "    switch(event.key) {\n";
  html += "      case 'ArrowUp':    case 'w':\n";
  html += "      case 'ArrowDown':  case 's':\n";
  html += "      case 'ArrowLeft':  case 'a':\n";
  html += "      case 'ArrowRight': case 'd':\n";
  html += "        sendCmd('stop'); break;\n";
  html += "    }\n";
  html += "  });\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Configure motor driver pins
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  
  stopRobot(); // Initialize stopped
  
  Serial.println("\nESP32 Drive Controller starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/drive", HTTP_POST, handlePostDrive);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N Driver**, and **2 DC Motors** (Left, Right) onto the canvas.
2. Wire IN1/IN2 to **GPIO12/GPIO14**, and IN3/IN4 to **GPIO27/GPIO26**. Wire the motor outputs of the L298N to the respective Left and Right motors.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Click and hold the D-Pad Forward arrow button. Verify that both simulated motor widgets spin forward (green clockwise/counterclockwise depending on mounting direction).
6. Release the mouse button. Verify that both motors stop.
7. Click inside the browser window and press the arrow keys on your keyboard to control the motors.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Drive] Action executed -> FORWARD
[Drive] Action executed -> STOP
```

## Expected Canvas Behavior
* Pressing/holding control buttons or keyboard keys spins the simulated motor widgets on the canvas in matching directions.
* Releasing buttons stops the motors immediately.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `digitalWrite(IN1, HIGH)` | Sets left motor direction. |
| `digitalWrite(IN3, HIGH)` | Sets right motor direction. |
| `onmousedown` / `onmouseup` | Web triggers to start movement on click and stop on release. |
| `document.addEventListener('keydown', ...)` | Listens for keyboard key presses in the browser window. |

## Hardware & Safety Concept: Differential Drive Steering and Common Grounds
* **Differential Steering**: Robots turn by driving motors at different speeds or in opposite directions.
  - Forward: both motors forward.
  - Spin Left: left motor reverse, right motor forward.
  - Spin Right: left motor forward, right motor reverse.
* **Common Ground**: DC motors draw high current spikes that can reset the ESP32. Always power motors from a separate power supply (e.g. battery pack), but connect the battery ground to the ESP32 ground pin to establish a shared reference.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display arrows showing the current movement direction.
2. **Warning buzzer**: Add a buzzer (GPIO 15) to beep when reversing (backup beeper).
3. **Wider Speed Control**: Add PWM speed control using LEDC (Project 21) driven by a speed slider on the webpage.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot spins instead of moving forward | Motor wires swapped | Swap the two wires connected to one of the DC motors |
| ESP32 resets when motors start | High current draw | Use a separate battery pack for the motor driver; do not power motors from the ESP32 |
| The webpage loads but keyboard keys do not work | Focus missing | Click inside the browser window to focus keyboard inputs before pressing keys |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [46 - ESP32 Web-based Relay switch control](../beginner/46-esp32-web-based-relay-switch-control.md)
- [101 - ESP32 Automatic Barrier Gate IoT Console](101-esp32-automatic-barrier-gate-iot-console.md)
- [122 - ESP32 WebSocket low-latency Drive Controller](122-esp32-websocket-low-latency-drive-controller.md) (Next project)
