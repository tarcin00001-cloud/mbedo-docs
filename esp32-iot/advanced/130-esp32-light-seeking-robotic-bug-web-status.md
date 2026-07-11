# 130 - Light Seeking Robotic Bug Web Status

Build an autonomous phototaxis-driven robot on the ESP32 that navigates toward light sources using two front-mounted photoresistors (Left LDR on GPIO 34, Right LDR on GPIO 35), drives two DC motors via an L298N driver, and hosts a web dashboard displaying sensor intensities and movement states.

## Goal
Learn how to read multiple analog inputs, implement comparative threshold-based differential steering logic, serve web dashboards, and compile JSON API endpoints.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Two LDR sensors are on GPIOs 34 and 35, and an L298N driver on GPIOs 27, 26, 25, and 33 controls the motors. The robot compares light levels on both sides. If the left sensor detects more light, it steers left; if the right sensor detects more light, it steers right. If both sensors detect low light (< 20%), the robot stops. A webpage displays LDR values and robot actions.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Photoresistors (LDR) | `photoresistor` | Yes (2 pieces) | Yes (2 pieces) |
| 10 kΩ Resistors (voltage dividers) | `resistor` | No | Yes (2 pieces) |
| L298N H-Bridge Motor Driver | `l298n` | Yes | Yes |
| DC Gear Motors | `motor_dc` | Yes (2 pieces) | Yes (2 pieces) |
| Shared power / Battery Pack | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Left LDR | Pin 1 / Pin 2 | GPIO34 / 3V3 | Yellow / Red | Left analog input |
| Right LDR | Pin 1 / Pin 2 | GPIO35 / 3V3 | Green / Red | Right analog input |
| L298N Driver | IN1 / IN2 | GPIO27 / GPIO26 | Blue / White | Left motor control |
| L298N Driver | IN3 / IN4 | GPIO25 / GPIO33 | Purple / Grey | Right motor control |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 10 kΩ resistor as a pull-down in series with each LDR to create a voltage divider circuit.

## Code
```cpp
// Light Seeking Robotic Bug Web Status (Phototaxis vehicle navigation + Telemetry Console)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Analog LDR pins
const int LEFT_LDR_PIN = 34;
const int RIGHT_LDR_PIN = 35;

// L298N motor driver direction pins
const int IN1 = 27;
const int IN2 = 26;
const int IN3 = 25;
const int IN4 = 33;

WebServer server(80);

int leftLight = 0;
int rightLight = 0;
String currentAction = "STOPPED";

// Drive control routines
void moveForward() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  currentAction = "FORWARD";
}

void turnLeft() {
  // Steer left toward light source
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  currentAction = "STEER LEFT";
}

void turnRight() {
  // Steer right toward light source
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  currentAction = "STEER RIGHT";
}

void stopRobot() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
  currentAction = "STOPPED (DARK)";
}

// HTTP API endpoint returning JSON data
void handleGetTelemetry() {
  String json = "{\"left\":" + String(leftLight) + 
                 ",\"right\":" + String(rightLight) + 
                 ",\"action\":\"" + currentAction + "\"}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Robotic Bug HUD</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .status-badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 14px; text-transform: uppercase; margin-bottom: 25px; background-color: #334155; color: #94a3b8; }\n";
  html += "  .status-badge.moving { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 20px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; }\n";
  html += "  .metric-label { font-size: 12px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 32px; font-weight: bold; margin-top: 5px; color: #38bdf8; font-family: monospace; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Robotic Bug Dashboard</h1>\n";
  html += "  <div id=\"actionDisplay\" class=\"status-badge\">STOPPED</div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Left Light</div><div class=\"metric-val\" id=\"leftDisplay\">0%</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Right Light</div><div class=\"metric-val\" id=\"rightDisplay\">0%</div></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Telemetry console active | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateBugHUD() {\n";
  html += "    fetch('/api/telemetry')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('leftDisplay').innerText = data.left + '%';\n";
  html += "        document.getElementById('rightDisplay').innerText = data.right + '%';\n";
  
  html += "        const bdgEl = document.getElementById('actionDisplay');\n";
  html += "        bdgEl.innerText = data.action;\n";
  
  html += "        if (data.action.startsWith('STOPPED')) {\n";
  html += "          bdgEl.className = 'status-badge';\n";
  html += "        } else {\n";
  html += "          bdgEl.className = 'status-badge moving';\n";
  html += "        }\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateBugHUD();\n";
  html += "    setInterval(updateBugHUD, 500); // Poll twice a second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LEFT_LDR_PIN, INPUT);
  pinMode(RIGHT_LDR_PIN, INPUT);
  
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  
  stopRobot();
  
  Serial.println("\nESP32 Light Seeking Bug Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/telemetry", handleGetTelemetry);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // 1. Read LDR sensors (0-4095) and map to percentages (0-100%)
  int rawLeft = analogRead(LEFT_LDR_PIN);
  int rawRight = analogRead(RIGHT_LDR_PIN);
  
  // Note: Depending on divider configuration, higher light may give higher voltage
  leftLight = map(rawLeft, 0, 4095, 0, 100);
  rightLight = map(rawRight, 0, 4095, 0, 100);
  
  int lightDifference = leftLight - rightLight;
  
  // 2. Comparative phototaxis logic
  // Sleep / Stop mode if it is too dark (< 20% on both sides)
  if (leftLight < 20 && rightLight < 20) {
    stopRobot();
  } 
  // Steer left if left sensor detects more light
  else if (lightDifference > 15) {
    turnLeft();
  } 
  // Steer right if right sensor detects more light
  else if (lightDifference < -15) {
    turnRight();
  } 
  // Proceed forward if light levels are balanced
  else {
    moveForward();
  }
  
  delay(30);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **2 Potentiometers** (to simulate Left/Right LDRs), **L298N Driver**, and **2 DC Motors** onto the canvas.
2. Wire Left Potentiometer to **GPIO34**, Right Potentiometer to **GPIO35**, and IN1–IN4 to **GPIO27, 26, 25, and 33**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Set both potentiometers to 50% (balanced). Verify that the motor widgets spin forward (green clockwise/counterclockwise).
6. Slide the Left potentiometer to 80% and Right to 20% (light on left). Verify that the left motor reverses and the right motor spins forward (steering left), and the webpage displays `STEER LEFT`.
7. Set both sliders to 5%. Verify that the motors stop.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
```

Browser JSON Console (`/api/telemetry`):
```json
{"left":82,"right":45,"action":"STEER LEFT"}
```

## Expected Canvas Behavior
* Adjusting the potentiometer sliders changes motor rotation directions to steer toward the simulated light source.
* The webpage display shows live updates of the light percentages.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `analogRead(LEFT_LDR_PIN)` | Samples the voltage of the left LDR voltage divider. |
| `leftLight < 20 && rightLight < 20` | Energy-saving check to stop the robot in the dark. |
| `lightDifference > 15` | Steers the robot left toward the stronger light source. |
| `moveForward()` | Drives the robot forward when light levels are balanced. |

## Hardware & Safety Concept: Sensor Calibration and Hysteresis Deadbands
* **Calibration**: Light sensors can vary due to manufacturing differences. To calibrate, place the robot under uniform light and adjust the software offset values in the code so both sensors read the same value (e.g. 50%).
* **Hysteresis Deadband**: Using a deadband threshold (e.g. `15%` difference) prevents the robot from oscillating rapidly left and right when the light levels are almost equal, protecting the motors.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a sun icon showing where the light is.
2. **Reverse Bug Alarm**: Add a buzzer (GPIO 15) to sound a chirp when the robot stops in the dark.
3. **SPIFFS logging**: Log average daily light readings to a file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot steers away from light | Steering logic inverted | Swap the `turnLeft()` and `turnRight()` calls in the code |
| Robot shakes left and right | Deadband too narrow | Increase the `lightDifference` threshold (e.g. from 15 to 20 or 25) |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [48 - ESP32 LDR light meter remote report](../beginner/48-esp32-ldr-light-meter-remote-report.md)
- [125 - ESP32 Line Following Robot with Web Configuration](125-esp32-line-following-robot-with-web-configuration-dashboard.md)
- [131 - ESP32 Sound Activated Mobile Robot Web Controls](131-esp32-sound-activated-mobile-robot-web-controls.md) (Next project)
