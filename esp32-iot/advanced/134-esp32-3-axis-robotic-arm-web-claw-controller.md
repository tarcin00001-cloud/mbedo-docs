# 134 - 3-Axis Robotic Arm Web claw controller

Build a web-controlled 3-axis robotic arm with a gripper claw on the ESP32 that drives four servo motors (Base on GPIO 12, Shoulder on GPIO 13, Elbow on GPIO 14, and Claw on GPIO 15) using the `ESP32Servo` library, and hosts a web dashboard with joint sliders and claw control buttons.

## Goal
Learn how to control multiple servo motors using `ESP32Servo`, map claw limits (open/closed states), build complex web panels, and parse multiple HTTP POST parameters.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Four servos on GPIOs 12, 13, 14, and 15 control a robotic arm and gripper claw. Navigating to the ESP32's IP address displays a control panel with three range sliders (Base, Shoulder, Elbow) and two buttons (Open Claw, Close Claw). Adjusting the interface controls the servos immediately (Claw open = 30°, closed = 120°).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Servo Motors | `servo` | Yes (4 pieces) | Yes (4 pieces) |
| Shared power / Battery Pack | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, monitor the four servo widgets to verify that they rotate in response to web interface adjustments.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Base Servo | PWM (Signal) | GPIO12 | Yellow | Rotation base axis |
| Shoulder Servo | PWM (Signal) | GPIO13 | Orange | Shoulder height axis |
| Elbow Servo | PWM (Signal) | GPIO14 | Green | Elbow reach axis |
| Claw Servo | PWM (Signal) | GPIO15 | Blue | Gripper claw actuator |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the servos from a separate 5V battery pack and connect its ground to the ESP32 ground pin.

## Code
```cpp
// 3-Axis Robotic Arm Web claw controller (Joint Angle Sliders + Gripper Toggle Buttons)
#include <WiFi.h>
#include <WebServer.h>
#include <ESP32Servo.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int BASE_PIN = 12;
const int SHLD_PIN = 13;
const int ELBW_PIN = 14;
const int CLAW_PIN = 15;

Servo baseServo;
Servo shoulderServo;
Servo elbowServo;
Servo clawServo;

WebServer server(80);

// Default coordinates
int baseAngle = 90;
int shoulderAngle = 90;
int elbowAngle = 90;
int clawAngle = 30; // 30 degrees = Open

bool clawOpened = true;

// HTTP API endpoint returning JSON data
void handleGetStatus() {
  String json = "{\"base\":" + String(baseAngle) + 
                 ",\"shoulder\":" + String(shoulderAngle) + 
                 ",\"elbow\":" + String(elbowAngle) + 
                 ",\"claw\":" + String(clawAngle) + 
                 ",\"clawOpen\":" + String(clawOpened ? 1 : 0) + "}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to update angles
void handlePostMove() {
  if (server.hasArg("base")) {
    baseAngle = server.arg("base").toInt();
    baseServo.write(baseAngle);
  }
  if (server.hasArg("shoulder")) {
    shoulderAngle = server.arg("shoulder").toInt();
    shoulderServo.write(shoulderAngle);
  }
  if (server.hasArg("elbow")) {
    elbowAngle = server.arg("elbow").toInt();
    elbowServo.write(elbowAngle);
  }
  
  server.send(200, "text/plain", "OK");
}

// HTTP POST endpoint to toggle claw
void handlePostClaw() {
  if (server.hasArg("state")) {
    String state = server.arg("state");
    if (state == "open") {
      clawAngle = 30; // Open limit
      clawOpened = true;
    } else if (state == "close") {
      clawAngle = 120; // Closed limit (grip)
      clawOpened = false;
    }
    clawServo.write(clawAngle);
    Serial.printf("[Claw Action] Gripper set to: %s (%d deg)\n", 
                  clawOpened ? "OPEN" : "CLOSED", clawAngle);
    server.send(200, "text/plain", "OK");
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Robotic Arm HUD</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .slider-container { margin: 20px 0; text-align: left; }\n";
  html += "  label { font-weight: bold; color: #94a3b8; display: block; margin-bottom: 8px; }\n";
  html += "  .slider-row { display: flex; align-items: center; gap: 15px; }\n";
  html += "  input[type=range] { flex-grow: 1; height: 8px; border-radius: 4px; background: #334155; outline: none; -webkit-appearance: none; }\n";
  html += "  input[type=range]::-webkit-slider-thumb { -webkit-appearance: none; width: 20px; height: 20px; border-radius: 50%; background: #38bdf8; cursor: pointer; }\n";
  html += "  .value-lbl { font-family: monospace; font-size: 16px; font-weight: bold; width: 45px; text-align: right; color: #10b981; }\n";
  html += "  .btn-group { display: flex; gap: 10px; margin-top: 25px; }\n";
  html += "  .btn { flex-grow: 1; padding: 12px; font-size: 14px; font-weight: bold; color: white; background-color: #334155; border: 1px solid #475569; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; }\n";
  html += "  .btn.active { background-color: #38bdf8; border-color: #0ea5e9; color: #0f172a; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Robotic Arm Console</h1>\n";
  
  // Base
  html += "  <div class=\"slider-container\">\n";
  html += "    <label for=\"baseSldr\">Base Angle</label>\n";
  html += "    <div class=\"slider-row\">\n";
  html += "      <input type=\"range\" id=\"baseSldr\" min=\"0\" max=\"180\" value=\"" + String(baseAngle) + "\" oninput=\"updateArm()\">\n";
  html += "      <span id=\"baseVal\" class=\"value-lbl\">" + String(baseAngle) + "&deg;</span>\n";
  html += "    </div>\n";
  html += "  </div>\n";
  
  // Shoulder
  html += "  <div class=\"slider-container\">\n";
  html += "    <label for=\"shldSldr\">Shoulder Angle</label>\n";
  html += "    <div class=\"slider-row\">\n";
  html += "      <input type=\"range\" id=\"shldSldr\" min=\"0\" max=\"180\" value=\"" + String(shoulderAngle) + "\" oninput=\"updateArm()\">\n";
  html += "      <span id=\"shldVal\" class=\"value-lbl\">" + String(shoulderAngle) + "&deg;</span>\n";
  html += "    </div>\n";
  html += "  </div>\n";
  
  // Elbow
  html += "  <div class=\"slider-container\">\n";
  html += "    <label for=\"elbwSldr\">Elbow Angle</label>\n";
  html += "    <div class=\"slider-row\">\n";
  html += "      <input type=\"range\" id=\"elbwSldr\" min=\"0\" max=\"180\" value=\"" + String(elbowAngle) + "\" oninput=\"updateArm()\">\n";
  html += "      <span id=\"elbwVal\" class=\"value-lbl\">" + String(elbowAngle) + "&deg;</span>\n";
  html += "    </div>\n";
  html += "  </div>\n";
  
  // Claw Buttons
  html += "  <h3>Gripper Claw Controls</h3>\n";
  html += "  <div class=\"btn-group\">\n";
  html += "    <button id=\"btnOpen\" class=\"btn\" onclick=\"setClaw('open')\">OPEN CLAW</button>\n";
  html += "    <button id=\"btnClose\" class=\"btn\" onclick=\"setClaw('close')\">CLOSE CLAW</button>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 1 second | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateDisplay() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('baseSldr').value = data.base;\n";
  html += "        document.getElementById('baseVal').innerHTML = data.base + '&deg;';\n";
  
  html += "        document.getElementById('shldSldr').value = data.shoulder;\n";
  html += "        document.getElementById('shldVal').innerHTML = data.shoulder + '&deg;';\n";
  
  html += "        document.getElementById('elbwSldr').value = data.elbow;\n";
  html += "        document.getElementById('elbwVal').innerHTML = data.elbow + '&deg;';\n";
  
  html += "        document.getElementById('btnOpen').className = 'btn' + (data.clawOpen === 1 ? ' active' : '');\n";
  html += "        document.getElementById('btnClose').className = 'btn' + (data.clawOpen === 0 ? ' active' : '');\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  function updateArm() {\n";
  html += "    const base = document.getElementById('baseSldr').value;\n";
  html += "    const shld = document.getElementById('shldSldr').value;\n";
  html += "    const elbw = document.getElementById('elbwSldr').value;\n";
  
  html += "    document.getElementById('baseVal').innerHTML = base + '&deg;';\n";
  html += "    document.getElementById('shldVal').innerHTML = shld + '&deg;';\n";
  html += "    document.getElementById('elbwVal').innerHTML = elbw + '&deg;';\n";
  
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('base', base);\n";
  html += "    body.append('shoulder', shld);\n";
  html += "    body.append('elbow', elbw);\n";
  
  html += "    fetch('/api/move', { method: 'POST', body: body });\n";
  html += "  }\n";
  
  html += "  function setClaw(stateVal) {\n";
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('state', stateVal);\n";
  html += "    fetch('/api/claw', { method: 'POST', body: body })\n";
  html += "      .then(() => updateDisplay());\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateDisplay();\n";
  html += "    setInterval(updateDisplay, 2000); // Poll every 2 seconds\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Attach Servos and move to default positions
  baseServo.attach(BASE_PIN);
  baseServo.write(baseAngle);
  
  shoulderServo.attach(SHLD_PIN);
  shoulderServo.write(shoulderAngle);
  
  elbowServo.attach(ELBW_PIN);
  elbowServo.write(elbowAngle);
  
  clawServo.attach(CLAW_PIN);
  clawServo.write(clawAngle);
  
  Serial.println("\nESP32 Claw Controller starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/status", HTTP_GET, handleGetStatus);
  server.on("/api/move", HTTP_POST, handlePostMove);
  server.on("/api/claw", HTTP_POST, handlePostClaw);
  
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
1. Drag **ESP32 DevKitC** and **4 Servo Motors** (Base, Shoulder, Elbow, Claw) onto the canvas.
2. Wire Base Servo to **GPIO12**, Shoulder to **GPIO13**, Elbow to **GPIO14**, and Claw to **GPIO15**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Click the "CLOSE CLAW" button on the webpage. Verify that the Claw servo widget on the canvas rotates to 120 degrees immediately.
6. Click "OPEN CLAW" to verify it returns to 30 degrees.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Claw Action] Gripper set to: CLOSED (120 deg)
[Claw Action] Gripper set to: OPEN (30 deg)
```

## Expected Canvas Behavior
* Clicking claw buttons on the webpage rotates the Claw servo widget on the canvas between 30° and 120° instantly.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `clawServo.attach(CLAW_PIN)` | Binds the gripper claw servo to GPIO 15. |
| `clawAngle = 30` | Sets the servo angle to open the gripper. |
| `clawAngle = 120` | Sets the servo angle to close the gripper and grip objects. |
| `server.arg("state")` | Extracts the claw state (open/close) from the HTTP POST payload. |

## Hardware & Safety Concept: Gripper Torque Limits and Mechanical Protection
* **Gripper Torque Limits**: Closing a gripper claw completely against a solid object (such as a plastic block) stalls the servo motor, drawing continuous high current and potentially melting its gears or casing.
* **Current Protection**: To protect the gripper mechanism, either configure the closed angle (`120°`) to be slightly wider than the object's width, or place a spring in series with the claw linkage to absorb excess travel and limit force.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a claw icon showing if it is open or closed.
2. **Alert sound**: Add a buzzer (GPIO 4) to play a beep when the claw opens or closes.
3. **SPIFFS integration**: Log claw operations to a file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The claw does not close completely | Angle limits wrong | Adjust the closed angle value in the code (e.g. from 120 to 140) to fit your hardware |
| Servos jump randomly when one moves | Ground loop noise | Ensure that all grounds (ESP32, servos, power supply) are connected together |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [56 - ESP32 Web Page Servo angle controller](56-esp32-web-page-servo-angle-controller-oled-display-feedback.md)
- [133 - ESP32 2-Axis Robotic Arm Web Coordinate balancer](133-esp32-2-axis-robotic-arm-web-coordinate-balancer.md)
- [135 - ESP32 Robotic Arm Web Position Memory Log](135-esp32-robotic-arm-web-position-memory-log.md) (Next project)
