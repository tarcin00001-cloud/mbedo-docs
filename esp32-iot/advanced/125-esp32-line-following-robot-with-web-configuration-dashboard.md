# 125 - Line Following Robot with Web Configuration dashboard

Build a web-configurable line following robot on the ESP32 that reads two photo-reflective infrared sensors (Left on GPIO 12, Right on GPIO 14), drives two DC motors via an L298N driver, and hosts a web dashboard with sliders to calibrate motor speed and steering sensitivity dynamically.

## Goal
Learn how to read digital infrared reflective sensors, generate hardware PWM using the ESP32 **LEDC** peripheral, implement closed-loop line-following logic, serve web pages, and process HTTP POST calibration updates.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Two IR reflective line sensors are on GPIOs 12 and 14. An L298N motor driver on GPIOs 27, 26, 25, and 33 controls the motors, with speed pins (ENA/ENB) on GPIOs 15 and 2. The robot follows a black line. Navigating to the ESP32's IP address displays a dashboard with sliders to configure base speed (PWM duty) and steering calibration.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| IR Reflective Line Sensors (TCRT5000) | `button` | Yes (2 pieces) | Yes (2 pieces) |
| L298N H-Bridge Motor Driver | `l298n` | Yes | Yes |
| DC Gear Motors | `motor_dc` | Yes (2 pieces) | Yes (2 pieces) |
| Shared power / Battery Pack | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use two pushbuttons to simulate the TCRT5000 line sensor outputs (pressed = on line/black, unpressed = clear/white).

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| TCRT5000 Sensor (L) | OUT (Digital) | GPIO12 | Yellow | Left line sensor input |
| TCRT5000 Sensor (R) | OUT (Digital) | GPIO14 | Green | Right line sensor input |
| L298N Driver | IN1 / IN2 | GPIO27 / GPIO26 | Blue / White | Left motor direction |
| L298N Driver | IN3 / IN4 | GPIO25 / GPIO33 | Purple / Grey | Right motor direction |
| L298N Driver | ENA / ENB | GPIO15 / GPIO2 | Orange / Black | PWM Speed control |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect speed control pins ENA and ENB of the L298N driver to the ESP32. Power the sensors and driver from the 5V Vin rail.

## Code
```cpp
// Line Following Robot with Web Configuration (Dual IR Line Tracker + LEDC Speed calibration)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// TCRT5000 IR reflective line sensor pins
const int LEFT_SENSOR_PIN = 12;
const int RIGHT_SENSOR_PIN = 14;

// L298N motor driver direction pins
const int IN1 = 27;
const int IN2 = 26;
const int IN3 = 25;
const int IN4 = 33;

// L298N PWM speed pins
const int ENA_PIN = 15;
const int ENB_PIN = 2;

// LEDC Hardware PWM configuration
const int PWM_FREQ = 5000;
const int PWM_RES = 8; // 8-bit resolution (0-255 speed steps)
const int PWM_CHAN_LEFT = 0;
const int PWM_CHAN_RIGHT = 1;

WebServer server(80);

// Configurable parameters via WiFi Web Console
int baseSpeed = 160;     // Default base PWM speed (0-255)
float steerGain = 1.2;   // Steering boost multiplier for sharp turns
String robotAction = "STOP";

// Reading cache
bool leftOnLine = false;
bool rightOnLine = false;

// Motor drive helper
void setMotors(int leftSpeed, int rightSpeed) {
  // Constrain speeds to 8-bit limit
  leftSpeed = constrain(leftSpeed, -255, 255);
  rightSpeed = constrain(rightSpeed, -255, 255);
  
  // Left Motor Direction
  if (leftSpeed >= 0) {
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
    ledcWrite(PWM_CHAN_LEFT, leftSpeed);
  } else {
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, HIGH);
    ledcWrite(PWM_CHAN_LEFT, -leftSpeed);
  }
  
  // Right Motor Direction
  if (rightSpeed >= 0) {
    digitalWrite(IN3, HIGH);
    digitalWrite(IN4, LOW);
    ledcWrite(PWM_CHAN_RIGHT, rightSpeed);
  } else {
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, HIGH);
    ledcWrite(PWM_CHAN_RIGHT, -rightSpeed);
  }
}

// HTTP API endpoint returning JSON data
void handleGetConfig() {
  String json = "{\"speed\":" + String(baseSpeed) + 
                 ",\"gain\":" + String(steerGain, 2) +
                 ",\"left\":" + String(leftOnLine ? 1 : 0) +
                 ",\"right\":" + String(rightOnLine ? 1 : 0) +
                 ",\"action\":\"" + robotAction + "\"}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to update calibration parameters
void handlePostConfigure() {
  if (server.hasArg("speed")) {
    baseSpeed = server.arg("speed").toInt();
  }
  if (server.hasArg("gain")) {
    steerGain = server.arg("gain").toFloat();
  }
  
  Serial.printf("[Calibrate] Base Speed: %d | Steering Gain: %.2f\n", baseSpeed, steerGain);
  server.send(200, "text/plain", "OK");
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Robot Calibration Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .status-badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 14px; text-transform: uppercase; margin: 15px 0; background-color: #38bdf8; color: #0f172a; }\n";
  
  // Visual IR sensors
  html += "  .sensor-row { display: flex; gap: 20px; justify-content: center; margin: 20px 0; }\n";
  html += "  .sensor-widget { display: flex; flex-direction: column; align-items: center; font-size: 12px; color: #64748b; }\n";
  html += "  .led-bulb { width: 24px; height: 24px; border-radius: 50%; background-color: #334155; border: 2px solid #475569; margin-top: 5px; transition: background-color 0.1s; }\n";
  html += "  .led-bulb.active { background-color: #38bdf8; box-shadow: 0 0 10px #38bdf8; }\n";
  
  html += "  .slider-container { margin: 20px 0; text-align: left; }\n";
  html += "  label { font-weight: bold; color: #94a3b8; display: block; margin-bottom: 8px; }\n";
  html += "  input[type=range] { width: 100%; height: 8px; border-radius: 4px; background: #334155; outline: none; -webkit-appearance: none; }\n";
  html += "  input[type=range]::-webkit-slider-thumb { -webkit-appearance: none; width: 20px; height: 20px; border-radius: 50%; background: #38bdf8; cursor: pointer; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Line Tracker Calibration</h1>\n";
  html += "  <div id=\"actionDisplay\" class=\"status-badge\">STOP</div>\n";
  
  // IR Sensors Feedback indicators
  html += "  <div class=\"sensor-row\">\n";
  html += "    <div class=\"sensor-widget\"><span>Left IR</span><div id=\"leftIrInd\" class=\"led-bulb\"></div></div>\n";
  html += "    <div class=\"sensor-widget\"><span>Right IR</span><div id=\"rightIrInd\" class=\"led-bulb\"></div></div>\n";
  html += "  </div>\n";
  
  // Sliders
  html += "  <div class=\"slider-container\">\n";
  html += "    <label for=\"spdRange\">Base Speed (PWM): <span id=\"spdVal\">160</span></label>\n";
  html += "    <input type=\"range\" id=\"spdRange\" min=\"80\" max=\"250\" value=\"" + String(baseSpeed) + "\" oninput=\"updateCalibrate()\">\n";
  html += "  </div>\n";
  
  html += "  <div class=\"slider-container\">\n";
  html += "    <label for=\"gainRange\">Steering Gain: <span id=\"gainVal\">1.20</span></label>\n";
  html += "    <input type=\"range\" id=\"gainRange\" min=\"1.0\" max=\"2.0\" step=\"0.1\" value=\"" + String(steerGain, 1) + "\" oninput=\"updateCalibrate()\">\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Telemetry configuration portal active | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateConsole() {\n";
  html += "    fetch('/api/config')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('actionDisplay').innerText = data.action;\n";
  html += "        document.getElementById('leftIrInd').className = 'led-bulb' + (data.left === 1 ? ' active' : '');\n";
  html += "        document.getElementById('rightIrInd').className = 'led-bulb' + (data.right === 1 ? ' active' : '');\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  function updateCalibrate() {\n";
  html += "    const spd = document.getElementById('spdRange').value;\n";
  html += "    const gn = document.getElementById('gainRange').value;\n";
  
  html += "    document.getElementById('spdVal').innerText = spd;\n";
  html += "    document.getElementById('gainVal').innerText = parseFloat(gn).toFixed(2);\n";
  
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('speed', spd);\n";
  html += "    body.append('gain', gn);\n";
  
  html += "    fetch('/api/configure', { method: 'POST', body: body });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateConsole();\n";
  html += "    setInterval(updateConsole, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Configure line sensors
  pinMode(LEFT_SENSOR_PIN, INPUT);
  pinMode(RIGHT_SENSOR_PIN, INPUT);
  
  // Configure motor direction pins
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  
  // Configure LEDC PWM Speed controller pins
  ledcSetup(PWM_CHAN_LEFT, PWM_FREQ, PWM_RES);
  ledcAttachPin(ENA_PIN, PWM_CHAN_LEFT);
  
  ledcSetup(PWM_CHAN_RIGHT, PWM_FREQ, PWM_RES);
  ledcAttachPin(ENB_PIN, PWM_CHAN_RIGHT);
  
  setMotors(0, 0); // Initialize stopped
  
  Serial.println("\nESP32 Line Tracker starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/config", handleGetConfig);
  server.on("/api/configure", HTTP_POST, handlePostConfigure);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // 1. Read reflective IR sensors (HIGH when on black line, LOW when on white)
  leftOnLine = (digitalRead(LEFT_SENSOR_PIN) == HIGH);
  rightOnLine = (digitalRead(RIGHT_SENSOR_PIN) == HIGH);
  
  // Calculate scaled steering speed
  int turnSpeed = (int)((float)baseSpeed * steerGain);
  
  // 2. Closed-Loop Line Following Logic
  if (leftOnLine && rightOnLine) {
    // Both on black line -> Stop (or junction cross)
    setMotors(0, 0);
    robotAction = "STOP";
  } 
  else if (leftOnLine && !rightOnLine) {
    // Left on line -> Turn left (spin left motor reverse, right motor forward)
    setMotors(-turnSpeed, turnSpeed);
    robotAction = "STEER LEFT";
  } 
  else if (!leftOnLine && rightOnLine) {
    // Right on line -> Turn right (spin left motor forward, right motor reverse)
    setMotors(turnSpeed, -turnSpeed);
    robotAction = "STEER RIGHT";
  } 
  else {
    // Both on white background -> Proceed Forward
    setMotors(baseSpeed, baseSpeed);
    robotAction = "FORWARD";
  }
  
  delay(5);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **2 Pushbuttons** (to simulate Left/Right line sensors), **L298N Driver**, and **2 DC Motors** onto the canvas.
2. Wire Left Button to **GPIO12**, Right Button to **GPIO14**, IN1–IN4 to **GPIO27, 26, 25, 33**, and ENA/ENB to **GPIO15/GPIO2**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. In normal state (unpressed buttons), verify that both simulated motors spin forward (green clockwise/counterclockwise).
6. Press the Left button widget (simulating left sensor hitting a line). Verify that the left motor reverses and the right motor spins forward.
7. Use the speed slider on the web page to increase the base speed. Verify that the motor widgets spin faster.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Calibrate] Base Speed: 180 | Steering Gain: 1.40
```

## Expected Canvas Behavior
* Pressing/holding the simulated line buttons alters motor rotation direction to follow the line.
* Adjusting webpage calibration sliders changes motor speed and steering boost instantly.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `ledcSetup(...)` / `ledcAttachPin(...)` | Sets up the hardware PWM channels to drive the motor speed pins. |
| `ledcWrite(...)` | Updates the duty cycle on the speed pin to adjust motor RPM. |
| `setMotors(-turnSpeed, turnSpeed)` | Drives left motor reverse and right motor forward to execute a pivot turn. |
| `server.arg("speed")` | Parses base speed calibration updates from web configuration console. |

## Hardware & Safety Concept: Sensor Hysteresis and Motor Calibration
* **Sensor Calibration**: IR sensors are highly sensitive to ambient light and tracking surface height. Most TCRT5000 modules include a built-in trimmer potentiometer. Turn the potentiometer until the onboard indicator LED turns ON when over a black line and OFF when over white paper.
* **PWM Speed Constraints**: DC motors require a minimum voltage to overcome friction (typically a PWM duty cycle > 80). If the base speed is configured below this threshold, the motors will stall and hum, potentially overheating. Set a minimum value in the webpage slider (e.g. `min="80"`) to prevent motor stall.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a matching status graphic.
2. **Alert buzzer beep**: Sound a short warning beep on a buzzer (GPIO 15) when the robot stops.
3. **SPIFFS integration**: Save base speed and steering gain values to SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot turns in the wrong direction | Sensor pins swapped | Swap the Left and Right sensor GPIO pin assignments in code |
| Motors hum but do not turn | PWM speed too low | Increase the base speed slider value on the webpage |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [49 - Potentiometer LED Bar Graph level controller](../beginner/49-potentiometer-led-bar-graph-level-controller.md)
- [121 - ESP32 WiFi Web Page Drive Controller](121-esp32-wifi-web-page-drive-controller.md)
- [124 - ESP32 Obstacle Avoidance Robot with Web HUD](124-esp32-obstacle-avoidance-robot-with-web-hud-telemetry.md)
