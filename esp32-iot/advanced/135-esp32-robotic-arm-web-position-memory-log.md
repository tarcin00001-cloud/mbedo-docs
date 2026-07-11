# 135 - Robotic Arm Web Position Memory Log (Save positions to ESP32 Flash)

Build a teachable robotic arm controller on the ESP32 that drives three servos (Base on GPIO 12, Shoulder on GPIO 13, Elbow on GPIO 14) using the `ESP32Servo` library, saves current joint coordinates to the ESP32's non-volatile SPIFFS flash filesystem, and hosts a web dashboard to adjust, save, and recall positions.

## Goal
Learn how to initialize the SPIFFS filesystem, write and read text files in flash memory, parse character buffers, drive multiple servo motors, and serve web interfaces.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Three servos on GPIOs 12, 13, and 14 control a robotic arm. Navigating to the ESP32's IP address displays a webpage with three sliders to adjust joint angles. The webpage features "Save Position" and "Load Position" buttons. Clicking Save writes the current coordinates to a file in SPIFFS. Clicking Load reads the saved coordinates and moves the servos to those angles. The saved position is restored automatically on reboot.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Servo Motors | `servo` | Yes (3 pieces) | Yes (3 pieces) |
| Shared power / Battery Pack | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, verify that the three servo widgets rotate to the saved angles when you click the Load button.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Base Servo | PWM (Signal) | GPIO12 | Yellow | Rotation base axis |
| Shoulder Servo | PWM (Signal) | GPIO13 | Orange | Shoulder height axis |
| Elbow Servo | PWM (Signal) | GPIO14 | Green | Elbow reach axis |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the servos from a separate 5V battery pack and connect its ground to the ESP32 ground pin to avoid resetting the board.

## Code
```cpp
// Robotic Arm Web Position Memory (Servo Angle controllers + SPIFFS Flash Saver)
#include <WiFi.h>
#include <WebServer.h>
#include <ESP32Servo.h>
#include <SPIFFS.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int BASE_PIN = 12;
const int SHLD_PIN = 13;
const int ELBW_PIN = 14;

Servo baseServo;
Servo shoulderServo;
Servo elbowServo;

WebServer server(80);

// Default coordinates
int baseAngle = 90;
int shoulderAngle = 90;
int elbowAngle = 90;

const char* filepath = "/position.txt";

// Save coordinates to SPIFFS flash memory
bool savePositionToFlash() {
  File file = SPIFFS.open(filepath, FILE_WRITE);
  if (!file) {
    Serial.println("[SPIFFS Error] Failed to open position file for writing!");
    return false;
  }
  
  // Format string: base,shoulder,elbow
  file.printf("%d,%d,%d\n", baseAngle, shoulderAngle, elbowAngle);
  file.close();
  Serial.printf("[SPIFFS] Saved positions: %d, %d, %d\n", baseAngle, shoulderAngle, elbowAngle);
  return true;
}

// Load coordinates from SPIFFS flash memory
bool loadPositionFromFlash() {
  if (!SPIFFS.exists(filepath)) {
    Serial.println("[SPIFFS] No saved position file found.");
    return false;
  }
  
  File file = SPIFFS.open(filepath, FILE_READ);
  if (!file) {
    Serial.println("[SPIFFS Error] Failed to open position file for reading!");
    return false;
  }
  
  String content = file.readStringUntil('\n');
  file.close();
  
  // Parse comma-separated variables
  int comma1 = content.indexOf(',');
  int comma2 = content.indexOf(',', comma1 + 1);
  
  if (comma1 != -1 && comma2 != -1) {
    baseAngle = content.substring(0, comma1).toInt();
    shoulderAngle = content.substring(comma1 + 1, comma2).toInt();
    elbowAngle = content.substring(comma2 + 1).toInt();
    
    // Bounds constraints
    baseAngle = constrain(baseAngle, 0, 180);
    shoulderAngle = constrain(shoulderAngle, 0, 180);
    elbowAngle = constrain(elbowAngle, 0, 180);
    
    // Move servos
    baseServo.write(baseAngle);
    shoulderServo.write(shoulderAngle);
    elbowServo.write(elbowAngle);
    
    Serial.printf("[SPIFFS] Loaded positions: %d, %d, %d\n", baseAngle, shoulderAngle, elbowAngle);
    return true;
  }
  
  return false;
}

// HTTP API endpoint returning JSON data
void handleGetStatus() {
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
  
  server.send(200, "text/plain", "OK");
}

// HTTP POST endpoint to save current position to flash
void handlePostSave() {
  if (savePositionToFlash()) {
    server.send(200, "text/plain", "Saved");
  } else {
    server.send(500, "text/plain", "Save Failed");
  }
}

// HTTP POST endpoint to load position from flash
void handlePostLoad() {
  if (loadPositionFromFlash()) {
    server.send(200, "text/plain", "Loaded");
  } else {
    server.send(500, "text/plain", "Load Failed");
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Teach Arm Console</title>\n";
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
  html += "  .btn-primary { background-color: #38bdf8; border-color: #0ea5e9; color: #0f172a; }\n";
  html += "  .btn-primary:hover { background-color: #0ea5e9; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Teachable Robotic Arm</h1>\n";
  
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
  
  // Memory Buttons
  html += "  <div class=\"btn-group\">\n";
  html += "    <button class=\"btn btn-primary\" onclick=\"savePos()\">Save Position</button>\n";
  html += "    <button class=\"btn\" onclick=\"loadPos()\">Load Position</button>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Positions saved to SPIFFS flash storage | Port 80</p>\n";
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
  
  html += "  function savePos() {\n";
  html += "    fetch('/api/save', { method: 'POST' })\n";
  html += "      .then(response => {\n";
  html += "        if(response.ok) alert('Position saved successfully!');\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  function loadPos() {\n";
  html += "    fetch('/api/load', { method: 'POST' })\n";
  html += "      .then(() => updateDisplay());\n";
  html += "  }\n";
  
  html += "  window.onload = updateDisplay;\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // 1. Initialize SPIFFS filesystem
  if (!SPIFFS.begin(true)) {
    Serial.println("[SPIFFS Error] Failed to mount filesystem!");
  } else {
    Serial.println("[SPIFFS] Filesystem mounted successfully.");
  }
  
  // Attach Servos
  baseServo.attach(BASE_PIN);
  shoulderServo.attach(SHLD_PIN);
  elbowServo.attach(ELBW_PIN);
  
  // 2. Try loading last saved coordinates on boot
  if (loadPositionFromFlash()) {
    Serial.println("[Boot] Restored position from SPIFFS flash.");
  } else {
    Serial.println("[Boot] No saved position found. Moving to default center.");
    baseServo.write(baseAngle);
    shoulderServo.write(shoulderAngle);
    elbowServo.write(elbowAngle);
  }
  
  Serial.println("\nESP32 Teachable Arm Server starting...");
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
  server.on("/api/save", HTTP_POST, handlePostSave);
  server.on("/api/load", HTTP_POST, handlePostLoad);
  
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
2. Wire Base Servo to **GPIO12**, Shoulder to **GPIO13**, and Elbow to **GPIO14**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Drag the Base slider to 30°, Shoulder to 150°, and Elbow to 60°.
6. Click the "Save Position" button. Verify that the console logs the saved values.
7. Move the sliders to different positions, then click "Load Position". Verify that the simulated servo widgets rotate back to the saved angles.
8. Reboot the ESP32 (stop and start simulation). Verify that the servos automatically restore to 30°, 150°, and 60° on boot.

## Expected Output
Serial Monitor:
```
[SPIFFS] Filesystem mounted successfully.
[SPIFFS] Loaded positions: 30, 150, 60
[Boot] Restored position from SPIFFS flash.
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[SPIFFS] Saved positions: 45, 120, 90
```

## Expected Canvas Behavior
* Clicking the "Load Position" button rotates the servo widgets to the stored SPIFFS configuration.
* The settings persist after stopping and starting the simulation.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `SPIFFS.begin(true)` | Mounts the SPIFFS filesystem, formatting it if corrupted. |
| `SPIFFS.open(..., FILE_WRITE)` | Opens a file handle to write data to flash. |
| `file.printf("%d,%d,%d\n", ...)` | Serializes the joint angles as a comma-separated string. |
| `loadPositionFromFlash()` | Reads the file, parses angles, and restores joint positions. |

## Hardware & Safety Concept: Flash Wear and Safe Shutdowns
* **Flash Wear (Write Limits)**: Flash memory (SPIFFS/LittleFS) has a limited lifespan (approx. 100,000 write cycles). Avoid writing coordinates to flash inside the main `loop()` continuously. Only write to flash when the user explicitly clicks the "Save" button.
* **Auto-Calibration**: On system startup, the arm should move slowly to the stored starting position to prevent rapid movements that could tip the robot over or damage the linkages.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a "Saved!" message when flash is updated.
2. **Sequence Player**: Extend the code to save a sequence of 5 different positions and play them back as an automated loop.
3. **Reset Button**: Add a physical button (GPIO 15) to format the SPIFFS filesystem and delete saved positions.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| SPIFFS failed to mount | Partition table error | Ensure the partition scheme in your board manager is configured with a SPIFFS block |
| Values are not saved | File open failure | Verify that the file path starts with a forward slash (e.g. `/position.txt`) |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [62 - ESP32 SPIFFS file logger](../intermediate/62-esp32-spiffs-file-logger-oled-diagnostics.md)
- [132 - ESP32 Servo Robotic Arm Web Slider controls](132-esp32-servo-robotic-arm-web-slider-controls.md)
- [136 - ESP32 SD Card Data Logger Web Viewer](136-esp32-sd-card-data-logger-web-viewer-host-directory-listing.md) (Next project)
