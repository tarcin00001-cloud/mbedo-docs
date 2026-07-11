# 161 - Solar Tracker Dual Axis Controller IoT Status Dashboard

Build a dual-axis solar panel tracking system on the ESP32 that reads four photoresistors (Top-Left on GPIO 34, Top-Right on GPIO 35, Bottom-Left on GPIO 32, Bottom-Right on GPIO 33) to measure light vectors, drives Pan and Tilt servos (on GPIOs 12 and 13) to steer toward the sun, and hosts a dashboard displaying live sensor values and servo angles.

## Goal
Learn how to implement comparative sensor array matrix logic, control multiple servo motors, apply angle safety constraints, build real-time web dashboards, and handle network integrations.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Four LDR sensors are arranged in a quadrant. Two servos on GPIOs 12 and 13 control a Pan/Tilt gimbal mechanism. The ESP32 compares light levels. If the top sensors detect more light than the bottom, it tilts upward; if the left sensors detect more light than the right, it pans left. A webpage displays LDR quadrant percentages and active gimbal angles.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Photoresistors (LDR) | `photoresistor` | Yes (4 pieces) | Yes (4 pieces) |
| 10 kΩ Resistors (voltage dividers) | `resistor` | No | Yes (4 pieces) |
| Servo Motors | `servo` | Yes (2 pieces) | Yes (2 pieces) |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use four potentiometers to simulate the Top-Left, Top-Right, Bottom-Left, and Bottom-Right LDR sensors.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR (Top-Left) | Pin 1 / Pin 2 | GPIO34 / 3V3 | Yellow / Red | Analog input |
| LDR (Top-Right) | Pin 1 / Pin 2 | GPIO35 / 3V3 | Orange / Red | Analog input |
| LDR (Bottom-Left)| Pin 1 / Pin 2 | GPIO32 / 3V3 | Green / Red | Analog input |
| LDR (Bottom-Right)| Pin 1 / Pin 2| GPIO33 / 3V3 | Blue / Red | Analog input |
| Pan Servo | PWM (Signal) | GPIO12 | Purple | Horizontal steering |
| Tilt Servo | PWM (Signal) | GPIO13 | Grey | Vertical steering |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power both servos from a separate 5V supply and connect its ground to the ESP32 ground pin.

## Code
```cpp
// Solar Tracker Dual Axis Controller IoT Status (4 LDR quadrant comparator + Pan/Tilt servos)
#include <WiFi.h>
#include <WebServer.h>
#include <ESP32Servo.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// LDR Pins
const int LDR_TL = 34; // Top Left
const int LDR_TR = 35; // Top Right
const int LDR_BL = 32; // Bottom Left
const int LDR_BR = 33; // Bottom Right

// Servo Pins
const int PAN_PIN = 12;
const int TILT_PIN = 13;

Servo panServo;
Servo tiltServo;

WebServer server(80);

// Servo angles
int panAngle = 90;
int tiltAngle = 90;

// Quadrant sensor readings (mapped to %)
int ldrTLVal = 0, ldrTRVal = 0, ldrBLVal = 0, ldrBRVal = 0;
String trackerStatus = "BALANCED";

// Execute quadrant comparison and adjust servo angles
void updateSolarTracker() {
  // Read analog values and map to percentage (0 - 100)
  ldrTLVal = map(analogRead(LDR_TL), 0, 4095, 0, 100);
  ldrTRVal = map(analogRead(LDR_TR), 0, 4095, 0, 100);
  ldrBLVal = map(analogRead(LDR_BL), 0, 4095, 0, 100);
  ldrBRVal = map(analogRead(LDR_BR), 0, 4095, 0, 100);
  
  // Calculate averages
  int avgTop = (ldrTLVal + ldrTRVal) / 2;
  int avgBottom = (ldrBLVal + ldrBRVal) / 2;
  int avgLeft = (ldrTLVal + ldrBLVal) / 2;
  int avgRight = (ldrTRVal + ldrBRVal) / 2;
  
  int diffV = avgTop - avgBottom; // Vertical difference
  int diffH = avgLeft - avgRight; // Horizontal difference
  
  const int deadband = 8; // Deadband to prevent servo jitter
  bool adjusting = false;
  
  // Vertical tracking (Tilt)
  if (abs(diffV) > deadband) {
    adjusting = true;
    if (diffV > 0) {
      tiltAngle--; // Tilt up
    } else {
      tiltAngle++; // Tilt down
    }
  }
  
  // Horizontal tracking (Pan)
  if (abs(diffH) > deadband) {
    adjusting = true;
    if (diffH > 0) {
      panAngle--; // Pan left
    } else {
      panAngle++; // Pan right
    }
  }
  
  // Constrain angles to mechanical limits (prevent cable strain/binding)
  panAngle = constrain(panAngle, 10, 170);
  tiltAngle = constrain(tiltAngle, 20, 160);
  
  // Drive servos
  panServo.write(panAngle);
  tiltServo.write(tiltAngle);
  
  if (adjusting) {
    trackerStatus = "TRACKING SUN";
  } else {
    trackerStatus = "BALANCED";
  }
}

// HTTP API endpoint returning JSON data
void handleGetStatus() {
  String json = "{\"tl\":" + String(ldrTLVal) + 
                 ",\"tr\":" + String(ldrTRVal) + 
                 ",\"bl\":" + String(ldrBLVal) + 
                 ",\"br\":" + String(ldrBRVal) + 
                 ",\"pan\":" + String(panAngle) + 
                 ",\"tilt\":" + String(tiltAngle) + 
                 ",\"status\":\"" + trackerStatus + "\"}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Solar Tracker HUD</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 480px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .status-badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 14px; text-transform: uppercase; margin-bottom: 25px; background-color: #334155; color: #94a3b8; }\n";
  html += "  .status-badge.tracking { background-color: #0369a1; color: #e0f2fe; }\n";
  
  // Quadrant matrix layout styles
  html += "  .quad-container { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; width: 200px; margin: 20px auto; }\n";
  html += "  .quad-cell { background-color: #0f172a; border: 1px solid #334155; border-radius: 8px; padding: 15px; font-family: monospace; font-size: 14px; }\n";
  html += "  .quad-label { font-size: 9px; color: #64748b; font-weight: bold; display: block; margin-bottom: 4px; }\n";
  
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin-top: 20px; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 12px; border: 1px solid #334155; text-align: left; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 22px; font-weight: bold; margin-top: 5px; color: #10b981; font-family: monospace; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Solar Tracker Dashboard</h1>\n";
  html += "  <div id=\"statusBadge\" class=\"status-badge\">BALANCED</div>\n";
  
  // LDR Quadrant grid
  html += "  <div class=\"quad-container\">\n";
  html += "    <div class=\"quad-cell\"><span class=\"quad-label\">TL</span><span id=\"tlDisplay\">0%</span></div>\n";
  html += "    <div class=\"quad-cell\"><span class=\"quad-label\">TR</span><span id=\"trDisplay\">0%</span></div>\n";
  html += "    <div class=\"quad-cell\"><span class=\"quad-label\">BL</span><span id=\"blDisplay\">0%</span></div>\n";
  html += "    <div class=\"quad-cell\"><span class=\"quad-label\">BR</span><span id=\"brDisplay\">0%</span></div>\n";
  html += "  </div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Pan Angle</div><div class=\"metric-val\" id=\"panDisplay\">90&deg;</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Tilt Angle</div><div class=\"metric-val\" id=\"tiltDisplay\">90&deg;</div></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Quadrant sensor logic loop | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateTrackerHUD() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('tlDisplay').innerText = data.tl + '%';\n";
  html += "        document.getElementById('trDisplay').innerText = data.tr + '%';\n";
  html += "        document.getElementById('blDisplay').innerText = data.bl + '%';\n";
  html += "        document.getElementById('brDisplay').innerText = data.br + '%';\n";
  html += "        document.getElementById('panDisplay').innerHTML = data.pan + '&deg;';\n";
  html += "        document.getElementById('tiltDisplay').innerHTML = data.tilt + '&deg;';\n";
  
  html += "        const bdgEl = document.getElementById('statusBadge');\n";
  html += "        bdgEl.innerText = data.status;\n";
  html += "        if(data.status === 'TRACKING SUN') {\n";
  html += "          bdgEl.className = 'status-badge tracking';\n";
  html += "        } else {\n";
  html += "          bdgEl.className = 'status-badge';\n";
  html += "        }\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateTrackerHUD();\n";
  html += "    setInterval(updateTrackerHUD, 500);\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Configure LDR inputs
  pinMode(LDR_TL, INPUT);
  pinMode(LDR_TR, INPUT);
  pinMode(LDR_BL, INPUT);
  pinMode(LDR_BR, INPUT);
  
  // Attach Servos and steer to center
  panServo.attach(PAN_PIN);
  panServo.write(panAngle);
  tiltServo.attach(TILT_PIN);
  tiltServo.write(tiltAngle);
  
  Serial.println("\nESP32 Solar Tracker starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/status", handleGetStatus);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // Run tracking calculations continuously
  updateSolarTracker();
  
  delay(20);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **2 Servos** (Pan, Tilt), and **4 Potentiometers** (to simulate Top-Left, Top-Right, Bottom-Left, and Bottom-Right LDR sensors) onto the canvas.
2. Wire Potentiometers to **GPIO34, 35, 32, and 33**.
3. Wire Pan Servo to **GPIO12** and Tilt Servo to **GPIO13**.
4. Paste the code and click **Run**.
5. Open your web browser and navigate to the printed IP address.
6. Set the Top-Left and Top-Right potentiometers to 80% (light from top). Verify that the Tilt servo widget rotates on the canvas.
7. Change the sliders so the Bottom-Right is high. Verify that both servos steer to align.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
```

Browser JSON Console (`/api/status`):
```json
{"tl":85,"tr":80,"bl":25,"br":30,"pan":75,"tilt":45,"status":"TRACKING SUN"}
```

## Expected Canvas Behavior
* Adjusting LDR sliders rotates the Pan and Tilt servo widgets on the canvas to steer toward the simulated light source.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `avgTop = ...` | Computes average light level on the top half of the quadrant sensor matrix. |
| `diffV = avgTop - avgBottom` | Calculates vertical error vector between top and bottom quadrants. |
| `panAngle--` / `tiltAngle++` | Steps the servo positions to align the panel with the light source. |
| `constrain(...)` | Enforces structural movement limits to protect the gimbal cables. |

## Hardware & Safety Concept: Servo Jitter Deadbands and Mechanical End-stops
* **Servo Jitter Deadbands**: Because analog readings fluctuate due to noise, the LDR values will differ slightly even in uniform light. Without a **deadband** (e.g. `deadband = 8`), the servos would jitter back and forth continuously, wearing out their gears and drawing excess current. Only adjust the servo when the difference exceeds this threshold.
* **Mechanical End-stops**: Solar trackers have cables connecting the panels to the base. Continuous rotation in one direction would twist and break these wires. Enforce software constraints (`constrain(panAngle, 10, 170)`) to restrict movement within a safe, non-twisting arc.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a sun icon showing where the tracker is pointing.
2. **Night Return Routine**: Implement a night routine: if all 4 LDRs read low light (< 15%) for more than 5 minutes, automatically return the servos to `90°` (facing east, ready for sunrise).
3. **SPIFFS tracker log**: Save average daily tracker positions to a file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servos steer away from light | Steering vectors inverted | Swap the increment (`++`) and decrement (`--`) statements for that axis in the code |
| Servos oscillate rapidly | Deadband too narrow | Increase the deadband value in the code (e.g., from 8 to 12 or 15) |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [130 - ESP32 Light Seeking Robotic Bug Web Status](130-esp32-light-seeking-robotic-bug-web-status.md)
- [133 - ESP32 2-Axis Robotic Arm Web Coordinate balancer](133-esp32-2-axis-robotic-arm-web-coordinate-balancer.md)
- [162 - Solar Tracker Efficiency Web Logger](162-esp32-solar-tracker-efficiency-web-logger-voltage-current-cloud-graph.md) (Next project)
