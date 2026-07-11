# 122 - WebSocket low-latency Drive Controller

Build a low-latency robotic drive controller on the ESP32 that runs WebSockets on port 81 for real-time navigation. It controls two DC motors using an L298N motor driver (IN1–IN4 on GPIOs 12, 14, 27, and 26) and hosts a styled control interface that transmits commands instantly.

## Goal
Learn how to establish low-latency WebSocket servers on the ESP32, stream short control payloads, interface L298N motor drivers, and build responsive client scripts.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network, hosting a web server on port 80 and a WebSocket listener on port 81. An L298N driver on GPIOs 12, 14, 27, and 26 drives two DC motors. Navigating to the ESP32's IP address displays a control panel. Clicking buttons or pressing the WASD/Arrow keys sends short direction characters over the WebSocket channel. This provides near-instantaneous motor response (under 10 ms delay).

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

> **Wiring tip:** Power the L298N driver and DC motors from a separate battery pack and connect the battery ground to the ESP32 ground pin.

## Code
```cpp
// WebSocket low-latency Drive Controller (Low-latency control link + Quad Motor drive)
#include <WiFi.h>
#include <WebServer.h>
#include <WebSocketsServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Left Motor Control Pins
const int IN1 = 12;
const int IN2 = 14;

// Right Motor Control Pins
const int IN3 = 27;
const int IN4 = 26;

WebServer server(80);
WebSocketsServer webSocket = WebSocketsServer(81);
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

// WebSocket Event Callback Handler
void webSocketEvent(uint8_t num, WStype_t type, uint8_t * payload, size_t length) {
  switch (type) {
    case WStype_DISCONNECTED:
      Serial.printf("[WebSocket] Client #%u disconnected.\n", num);
      stopRobot(); // Fail-safe: stop robot if web connection breaks
      break;
      
    case WStype_CONNECTED:
      Serial.printf("[WebSocket] Client #%u connected.\n", num);
      break;
      
    case WStype_TEXT: {
      char cmdChar = (char)payload[0];
      
      // Process fast direction command codes
      switch (cmdChar) {
        case 'F': moveForward();  break;
        case 'B': moveBackward(); break;
        case 'L': turnLeft();     break;
        case 'R': turnRight();    break;
        case 'S': stopRobot();    break;
      }
      
      // Send feedback status to the client
      String feedback = "DIR:" + currentDirection;
      webSocket.sendTXT(num, feedback.c_str());
      break;
    }
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>WebSocket Controller</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 400px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .status-badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 14px; text-transform: uppercase; margin: 15px 0; background-color: #334155; color: #94a3b8; }\n";
  html += "  .status-badge.connected { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .control-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 10px; margin: 25px auto; max-width: 260px; }\n";
  html += "  .btn { padding: 18px; font-size: 18px; font-weight: bold; color: white; background-color: #334155; border: 1px solid #475569; border-radius: 12px; cursor: pointer; transition: background-color 0.1s, transform 0.1s; }\n";
  html += "  .btn:active { background-color: #38bdf8; color: #0f172a; transform: scale(0.95); }\n";
  html += "  .btn-stop { background-color: #991b1b; border-color: #b91c1c; }\n";
  html += "  .btn-stop:hover { background-color: #b91c1c; }\n";
  html += "  .empty { visibility: hidden; }\n";
  html += "  .keyboard-tips { font-size: 11px; color: #64748b; margin-top: 20px; line-height: 1.4; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Low-Latency Control</h1>\n";
  html += "  <div id=\"connBadge\" class=\"status-badge\">DISCONNECTED</div>\n";
  html += "  <h3 id=\"dirDisplay\" style=\"margin: 10px 0; color: #10b981;\">DIR: STOP</h3>\n";
  
  // D-Pad Grid
  html += "  <div class=\"control-grid\">\n";
  html += "    <div class=\"empty\"></div>\n";
  html += "    <button class=\"btn\" onmousedown=\"sendCmd('F')\" onmouseup=\"sendCmd('S')\">&#9650;</button>\n";
  html += "    <div class=\"empty\"></div>\n";
  
  html += "    <button class=\"btn\" onmousedown=\"sendCmd('L')\" onmouseup=\"sendCmd('S')\">&#9664;</button>\n";
  html += "    <button class=\"btn btn-stop\" onclick=\"sendCmd('S')\">STOP</button>\n";
  html += "    <button class=\"btn\" onmousedown=\"sendCmd('R')\" onmouseup=\"sendCmd('S')\">&#9654;</button>\n";
  
  html += "    <div class=\"empty\"></div>\n";
  html += "    <button class=\"btn\" onmousedown=\"sendCmd('B')\" onmouseup=\"sendCmd('S')\">&#9660;</button>\n";
  html += "    <div class=\"empty\"></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"keyboard-tips\">WS Connection on Port 81<br>Use **WASD** or **Arrow Keys** on keyboard.</p>\n";
  html += "</div>\n";
  
  // Client-side WebSocket script
  html += "<script>\n";
  html += "  let ws;\n";
  html += "  let lastSentChar = 'S';\n";
  
  html += "  function initWebSocket() {\n";
  html += "    ws = new WebSocket('ws://' + window.location.hostname + ':81/');\n";
  
  html += "    ws.onopen = function() {\n";
  html += "      const bdg = document.getElementById('connBadge');\n";
  html += "      bdg.innerText = 'CONNECTED';\n";
  html += "      bdg.className = 'status-badge connected';\n";
  html += "    };\n";
  
  html += "    ws.onclose = function() {\n";
  html += "      const bdg = document.getElementById('connBadge');\n";
  html += "      bdg.innerText = 'DISCONNECTED';\n";
  html += "      bdg.className = 'status-badge';\n";
  html += "      setTimeout(initWebSocket, 2000); // Attempt reconnection every 2 seconds\n";
  html += "    };\n";
  
  html += "    ws.onmessage = function(evt) {\n";
  html += "      if (evt.data.startsWith('DIR:')) {\n";
  html += "        document.getElementById('dirDisplay').innerText = evt.data;\n";
  html += "      }\n";
  html += "    };\n";
  html += "  }\n";
  
  html += "  function sendCmd(charVal) {\n";
  html += "    if (ws && ws.readyState === WebSocket.OPEN) {\n";
  html += "      if (charVal !== lastSentChar) {\n";
  html += "        ws.send(charVal);\n";
  html += "        lastSentChar = charVal;\n";
  html += "      }\n";
  html += "    }\n";
  html += "  }\n";
  
  // Keyboard listeners
  html += "  document.addEventListener('keydown', function(event) {\n";
  html += "    if (event.repeat) return;\n";
  html += "    switch(event.key) {\n";
  html += "      case 'ArrowUp':    case 'w': sendCmd('F'); break;\n";
  html += "      case 'ArrowDown':  case 's': sendCmd('B'); break;\n";
  html += "      case 'ArrowLeft':  case 'a': sendCmd('L'); break;\n";
  html += "      case 'ArrowRight': case 'd': sendCmd('R'); break;\n";
  html += "    }\n";
  html += "  });\n";
  
  html += "  document.addEventListener('keyup', function(event) {\n";
  html += "    switch(event.key) {\n";
  html += "      case 'ArrowUp':    case 'w':\n";
  html += "      case 'ArrowDown':  case 's':\n";
  html += "      case 'ArrowLeft':  case 'a':\n";
  html += "      case 'ArrowRight': case 'd':\n";
  html += "        sendCmd('S'); break;\n";
  html += "    }\n";
  html += "  });\n";
  
  html += "  window.onload = initWebSocket;\n";
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
  
  Serial.println("\nESP32 WebSocket Drive Station Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.begin();
  
  // Start WebSocket Server
  webSocket.begin();
  webSocket.onEvent(webSocketEvent);
  
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  webSocket.loop();
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N Driver**, and **2 DC Motors** onto the canvas.
2. Wire IN1/IN2 to **GPIO12/GPIO14**, and IN3/IN4 to **GPIO27/GPIO26**. Wire the motor outputs of the L298N to the Left and Right motors.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Click and hold the D-Pad Forward arrow. Verify that both simulated motor widgets spin forward immediately, and the feedback shows `DIR: FORWARD`.
6. Press the Arrow keys on your keyboard to control the robot. Verify that the response is immediate and lag-free compared to HTTP POST control (Project 121).

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[WebSocket] Client #0 connected.
```

## Expected Canvas Behavior
* Pressing/holding control buttons or keyboard keys spins the simulated motor widgets on the canvas in matching directions.
* Releasing buttons stops the motors immediately.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `webSocket.begin()` | Starts the WebSocket listener on port 81. |
| `webSocket.onEvent(...)` | Registers the event handler callback function. |
| `WStype_TEXT` | Fired when a client sends a text payload over the socket connection. |
| `webSocket.sendTXT(...)` | Sends instant text feedback to the active web client. |

## Hardware & Safety Concept: Network Fail-Safes and Connection Loss Protection
Mobile robots present a collision hazard if they lose connection while moving. If a robot is driving forward at full speed and the WiFi link drops (e.g. the robot drives out of range), an HTTP system might continue moving forward forever because it never receives a "stop" command. WebSockets address this:
1. **Heartbeat Fail-Safe**: If a client disconnects, the WebSocket server fires a `WStype_DISCONNECTED` event. The code must handle this by calling `stopRobot()` immediately to shut down the motors.
2. **Ping-Pong Timeout**: Enable the WebSocket ping/pong heartbeat feature to detect link drops within 2 seconds even if no commands are being sent.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the number of connected WebSocket clients.
2. **Reverse Indicator**: Add a buzzer (GPIO 15) to sound a warning beep when the reverse command (`B`) is active.
3. **Speed Controller**: Implement speed levels (0–255) sent as a suffix (e.g. `F:200` to drive forward at speed 200) using `ledcWrite`.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot does not respond to web controls | Port blocking | Confirm that your local network allows traffic on port 81 |
| The webpage shows "DISCONNECTED" | Handshake failed | Verify that the WebSocket server is active and the client script points to the correct IP address |
| Motors spin backwards | Wiring reversed | Swap the output wires on the motor driver for that motor |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [72 - ESP32 WebSocket LED toggle low latency feedback](72-esp32-websocket-led-toggle-low-latency-feedback.md)
- [121 - ESP32 WiFi Web Page Drive Controller](121-esp32-wifi-web-page-drive-controller.md)
- [123 - ESP32-CAM WiFi Video Streamer Vehicle](123-esp32-cam-wifi-video-streamer-vehicle.md) (Next project)
