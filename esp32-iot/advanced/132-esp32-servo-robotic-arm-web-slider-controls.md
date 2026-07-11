# 132 - Servo Robotic Arm Web Slider controls

Build a web-controlled 3-axis robotic arm system on the ESP32 that drives three servo motors representing Base, Shoulder, and Elbow joints (on GPIOs 12, 13, and 14 respectively) using the `ESP32Servo` library, and hosts a web dashboard with three angle adjustment sliders.

## Goal
Learn how to control multiple servo motors using `ESP32Servo`, map degrees to hardware pulse widths, build styled web panels with multiple range sliders, and parse multi-parameter HTTP POST control values.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Three servo motors are connected to GPIOs 12, 13, and 14. Navigating to the ESP32's IP address displays a control panel with three sliders (Base, Shoulder, Elbow). Adjusting the sliders sends POST requests to the ESP32, which rotates the corresponding servos immediately (from 0 to 180 degrees).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Servo Motors | `servo` | Yes (3 pieces) | Yes (3 pieces) |
| Shared power / Battery Pack | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, monitor the three servo widgets to verify that they rotate in response to web slider adjustments.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Base Servo | PWM (Signal) | GPIO12 | Yellow | Rotation base axis |
| Shoulder Servo | PWM (Signal) | GPIO13 | Orange | Shoulder height axis |
| Elbow Servo | PWM (Signal) | GPIO14 | Green | Elbow reach axis |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Servo motors draw significant current during motion. Power all three servos from a separate 5V battery pack and connect its ground to the ESP32 ground pin.

## Code
```cpp
// Servo Robotic Arm Web Slider controls (Multi-axis Servo Controller)
#include <WiFi.h>
#include <WebServer.h>
#include <ESP32Servo.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int BASE_PIN = 12;
const int SHLD_PIN = 13;
const int ELBW_PIN = 14;

Servo baseServo;
Servo shoulderServo;
Servo elbowServo;

WebServer server(80);

// Servo angle memory (Default middle positions)
int baseAngle = 90;
int shoulderAngle = 90;
int elbowAngle = 90;

// HTTP API endpoint returning JSON data
void handleGetPosition() {
  String json = "{\"base\":" + String(baseAngle) + 
                 ",\"shoulder\":" + String(shoulderAngle) + 
                 ",\"elbow\":" + String(elbowAngle) + "}";
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
  
  Serial.printf("[Arm Move] Base: %d | Shoulder: %d | Elbow: %d\n", 
                baseAngle, shoulderAngle, elbowAngle);
  server.send(200, "text/plain", "OK");
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Robotic Arm Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .slider-container { margin: 25px 0; text-align: left; }\n";
  html += "  label { font-weight: bold; color: #94a3b8; display: block; margin-bottom: 8px; }\n";
  html += "  .slider-row { display: flex; align-items: center; gap: 15px; }\n";
  html += "  input[type=range] { flex-grow: 1; height: 8px; border-radius: 4px; background: #334155; outline: none; -webkit-appearance: none; }\n";
  html += "  input[type=range]::-webkit-slider-thumb { -webkit-appearance: none; width: 20px; height: 20px; border-radius: 50%; background: #38bdf8; cursor: pointer; }\n";
  html += "  .value-lbl { font-family: monospace; font-size: 16px; font-weight: bold; width: 45px; text-align: right; color: #10b981; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Robotic Arm Console</h1>\n";
  
  // Base Slider
  html += "  <div class=\"slider-container\">\n";
  html += "    <label for=\"baseSldr\">Base Axis Angle</label>\n";
  html += "    <div class=\"slider-row\">\n";
  html += "      <input type=\"range\" id=\"baseSldr\" min=\"0\" max=\"180\" value=\"" + String(baseAngle) + "\" oninput=\"updateArm()\">\n";
  html += "      <span id=\"baseVal\" class=\"value-lbl\">" + String(baseAngle) + "&deg;</span>\n";
  html += "    </div>\n";
  html += "  </div>\n";
  
  // Shoulder Slider
  html += "  <div class=\"slider-container\">\n";
  html += "    <label for=\"shldSldr\">Shoulder Axis Angle</label>\n";
  html += "    <div class=\"slider-row\">\n";
  html += "      <input type=\"range\" id=\"shldSldr\" min=\"0\" max=\"180\" value=\"" + String(shoulderAngle) + "\" oninput=\"updateArm()\">\n";
  html += "      <span id=\"shldVal\" class=\"value-lbl\">" + String(shoulderAngle) + "&deg;</span>\n";
  html += "    </div>\n";
  html += "  </div>\n";
  
  // Elbow Slider
  html += "  <div class=\"slider-container\">\n";
  html += "    <label for=\"elbwSldr\">Elbow Axis Angle</label>\n";
  html += "    <div class=\"slider-row\">\n";
  html += "      <input type=\"range\" id=\"elbwSldr\" min=\"0\" max=\"180\" value=\"" + String(elbowAngle) + "\" oninput=\"updateArm()\">\n";
  html += "      <span id=\"elbwVal\" class=\"value-lbl\">" + String(elbowAngle) + "&deg;</span>\n";
  html += "    </div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 1 second | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateDisplay() {\n";
  html += "    fetch('/api/position')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('baseSldr').value = data.base;\n";
  html += "        document.getElementById('baseVal').innerHTML = data.base + '&deg;';\n";
  
  html += "        document.getElementById('shldSldr').value = data.shoulder;\n";
  html += "        document.getElementById('shldVal').innerHTML = data.shoulder + '&deg;';\n";
  
  html += "        document.getElementById('elbwSldr').value = data.elbow;\n";
  html += "        document.getElementById('elbwVal').innerHTML = data.elbow + '&deg;';\n";
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
  
  html += "  window.onload = function() {\n";
  html += "    updateDisplay();\n";
  // We don't poll too fast to avoid UI jumps while sliding
  html += "    setInterval(updateDisplay, 2000);\n";
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
  
  Serial.println("\nESP32 Robotic Arm Server starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/position", HTTP_GET, handleGetPosition);
  server.on("/api/move", HTTP_POST, handlePostMove);
  
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
1. Drag **ESP32 DevKitC** and **3 Servo Motors** onto the canvas.
2. Wire Base Servo signal to **GPIO12**, Shoulder Servo to **GPIO13**, and Elbow Servo to **GPIO14**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Drag the Base slider to 45 degrees. Verify that the corresponding simulated Base servo widget rotates on the canvas.
6. Drag the Shoulder slider to 120 degrees. Verify that the Shoulder servo rotates.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Arm Move] Base: 45 | Shoulder: 90 | Elbow: 90
[Arm Move] Base: 45 | Shoulder: 120 | Elbow: 90
```

## Expected Canvas Behavior
* Dragging sliders on the webpage rotates the matching servo widgets on the canvas instantly.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `baseServo.attach(BASE_PIN)` | Configures the PWM channel and binds the control pin. |
| `baseServo.write(baseAngle)` | Sends the width pulse to position the servo shaft. |
| `server.arg("base")` | Extracts the base angle parameter from the HTTP POST payload. |

## Hardware & Safety Concept: Current Spikes and Servo Decoupling
* **Current Spikes**: Servos contain internal motors and gearboxes. When starting to move, they draw a large surge current (up to 1A each). Driving three servos simultaneously from the ESP32's 3.3V or 5V pin can cause the ESP32 to brown out and reset. Always use an external power supply to power the servos, connecting its ground to the ESP32 ground pin.
* **Decoupling Capacitors**: Place a large electrolytic capacitor (e.g. 470 µF or 1000 µF) across the servo power rail close to the servo connectors to absorb voltage drops and stabilize the system.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a coordinate map of the arm.
2. **Speed Limiting**: Modify the servo write logic to slowly transition between angles rather than jumping instantly, preventing mechanical stress.
3. **Preset positions**: Add button shortcuts on the webpage (e.g. "Pick Up", "Drop Off", "Home") that move all three servos to preset coordinates.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servos vibrate or make buzzing noises | Stalled or low power | Check that your power supply can source enough current (at least 2A) |
| Servos jump randomly when one moves | Ground loop interference | Ensure that all grounds (ESP32, servos, external power supply) are connected together |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [56 - ESP32 Web Page Servo angle controller](56-esp32-web-page-servo-angle-controller-oled-display-feedback.md)
- [131 - ESP32 Sound Activated Mobile Robot Web Controls](131-esp32-sound-activated-mobile-robot-web-controls.md)
- [133 - ESP32 2-Axis Robotic Arm Web Coordinate balancer](133-esp32-2-axis-robotic-arm-web-coordinate-balancer.md) (Next project)
