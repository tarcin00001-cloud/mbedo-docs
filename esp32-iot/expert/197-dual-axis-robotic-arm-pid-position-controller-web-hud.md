# 197 - Dual Axis Robotic Arm PID Position Controller Web HUD

Build a high-precision closed-loop robotic joint controller on the ESP32 that controls two servo motor axes (Pan on GPIO 13, Tilt on GPIO 12) using a PID algorithm, reads angular feedback (analog potentiometer and I2C MPU-6050), and streams setpoint vs. actual graphs via WebSockets.

## Goal
Learn how to program closed-loop control systems, write custom proportional-integral-derivative (PID) controllers, read MPU-6050 tilt angles, map analog feedback, and tune parameters online.

## What You Will Build
An ESP32 DevKitC acts as a robotic joint controller. It controls a Pan Servo (GPIO 13) and a Tilt Servo (GPIO 12). Pan feedback is read from a potentiometer on GPIO 34. Tilt feedback is read from the pitch angle of an MPU-6050 sensor. The user inputs target angles on a web dashboard. The ESP32 runs a PID loop on each joint to drive the servos until the sensors match the targets, graphing actual vs. setpoint angles in real-time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 2x Servo Motors | `servo` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| MPU-6050 Accelerometer/Gyro | `mpu6050` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Pan Servo | PWM (Signal) | GPIO13 | Orange | Pan joint control |
| Tilt Servo | PWM (Signal) | GPIO12 | Yellow | Tilt joint control |
| Potentiometer | WIPER (Signal) | GPIO34 | Green | Pan feedback analog input |
| MPU-6050 | SDA / SCL | GPIO21 / GPIO22 | Purple / Brown | Tilt feedback I2C bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Run separate 5V lines for the servo motors to prevent power fluctuations from resetting the ESP32.

## Code
```cpp
// Robotic Joint PID Controller (MPU-6050 Pitch + Potentiometer Pan + 2x Servos + WebSockets HUD)
#include <WiFi.h>
#include <WebServer.h>
#include <WebSocketsServer.h>
#include <Wire.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <ESP32Servo.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

Adafruit_MPU6050 mpu;
Servo panServo;
Servo tiltServo;

const int PAN_SERVO_PIN = 13;
const int TILT_SERVO_PIN = 12;
const int PAN_POT_PIN = 34;

WebServer server(80);
WebSocketsServer webSocket(81);

// Joint Setpoints (Target Angles 0-180)
float panTarget = 90.0;
float tiltTarget = 90.0;

// Joint Actual Angles
float panActual = 90.0;
float tiltActual = 90.0;

// PID Parameters (Default coefficients)
float Kp = 1.2;
float Ki = 0.05;
float Kd = 0.2;

// PID State variables
float panErrorPrior = 0.0, panIntegral = 0.0;
float tiltErrorPrior = 0.0, tiltIntegral = 0.0;

unsigned long lastTime = 0;

// Execute PID calculation loops for both joints
void updateJointPIDs(float dt) {
  if (dt <= 0.0) return;
  
  // 1. Pan Joint PID (Feedback: Potentiometer 0-180 degrees)
  int rawPot = analogRead(PAN_POT_PIN);
  panActual = (rawPot / 4095.0) * 180.0;
  
  float panError = panTarget - panActual;
  panIntegral += (panError * dt);
  float panDerivative = (panError - panErrorPrior) / dt;
  float panOutput = (Kp * panError) + (Ki * panIntegral) + (Kd * panDerivative);
  panErrorPrior = panError;
  
  // Drive Pan Servo based on PID output
  int panWriteVal = constrain(90 + panOutput, 0, 180);
  panServo.write(panWriteVal);
  
  // 2. Tilt Joint PID (Feedback: MPU-6050 Pitch Angle converted to 0-180 scale)
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);
  
  // Convert acceleration values to pitch angle in degrees (-90 to +90)
  float pitchRad = atan2(-a.acceleration.x, sqrt(a.acceleration.y * a.acceleration.y + a.acceleration.z * a.acceleration.z));
  float pitchDeg = pitchRad * 180.0 / PI;
  // Map -90..+90 to 0..180 servo scale
  tiltActual = pitchDeg + 90.0;
  
  // Mock simulation fallback if MPU-6050 is static
  if (rawPot == 0 && a.acceleration.x == 0.0) {
    // Simulated gradual response curve
    panActual += (panTarget - panActual) * 0.1;
    tiltActual += (tiltTarget - tiltActual) * 0.1;
  }
  
  float tiltError = tiltTarget - tiltActual;
  tiltIntegral += (tiltError * dt);
  float tiltDerivative = (tiltError - tiltErrorPrior) / dt;
  float tiltOutput = (Kp * tiltError) + (Ki * tiltIntegral) + (Kd * tiltDerivative);
  tiltErrorPrior = tiltError;
  
  // Drive Tilt Servo based on PID output
  int tiltWriteVal = constrain(90 + tiltOutput, 0, 180);
  tiltServo.write(tiltWriteVal);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Robotic PID Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 620px; width: 100%; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; text-align: center; margin-top: 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 20px 0; }\n";
  html += "  .control-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; }\n";
  html += "  label { display: block; font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; margin-bottom: 8px; }\n";
  html += "  .slider { width: 100%; height: 6px; border-radius: 3px; background: #334155; outline: none; -webkit-appearance: none; margin-bottom: 10px; }\n";
  html += "  .val-disp { font-family: monospace; font-size: 16px; font-weight: bold; color: #10b981; text-align: right; }\n";
  html += "  canvas { width: 100%; height: 180px; background-color: #0f172a; border-radius: 8px; border: 1px solid #334155; margin-top: 20px; }\n";
  html += "  .footer { text-align: center; margin-top: 25px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Robotic PID Balancer</h1>\n";
  
  // Setpoint Sliders
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"control-card\">\n";
  html += "      <label>Pan Joint (Target)</label>\n";
  html += "      <input type=\"range\" id=\"panIn\" class=\"slider\" min=\"10\" max=\"170\" value=\"90\" oninput=\"sendTarget()\">\n";
  html += "      <div class=\"val-disp\" id=\"panDisp\">90 &deg;</div>\n";
  html += "    </div>\n";
  html += "    <div class=\"control-card\">\n";
  html += "      <label>Tilt Joint (Target)</label>\n";
  html += "      <input type=\"range\" id=\"tiltIn\" class=\"slider\" min=\"10\" max=\"170\" value=\"90\" oninput=\"sendTarget()\">\n";
  html += "      <div class=\"val-disp\" id=\"tiltDisp\">90 &deg;</div>\n";
  html += "    </div>\n";
  html += "  </div>\n";
  
  // PID Coefficient sliders
  html += "  <h3>PID Coefficient Tuning</h3>\n";
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"control-card\">\n";
  html += "      <label>Proportional (Kp)</label>\n";
  html += "      <input type=\"range\" id=\"kpIn\" class=\"slider\" min=\"0.1\" max=\"5.0\" step=\"0.1\" value=\"" + String(Kp, 1) + "\" oninput=\"sendTuning()\">\n";
  html += "      <div class=\"val-disp\" id=\"kpDisp\">" + String(Kp, 1) + "</div>\n";
  html += "    </div>\n";
  html += "    <div class=\"control-card\">\n";
  html += "      <label>Derivative (Kd)</label>\n";
  html += "      <input type=\"range\" id=\"kdIn\" class=\"slider\" min=\"0.0\" max=\"2.0\" step=\"0.1\" value=\"" + String(Kd, 1) + "\" oninput=\"sendTuning()\">\n";
  html += "      <div class=\"val-disp\" id=\"kdDisp\">" + String(Kd, 1) + "</div>\n";
  html += "    </div>\n";
  html += "  </div>\n";
  
  html += "  <canvas id=\"chart\"></canvas>\n";
  
  html += "  <p class=\"footer\">I2C: MPU-6050 (Tilt) | ADC: Pot (Pan) | WebSocket: 81</p>\n";
  html += "</div>\n";
  
  // WebSockets and Canvas logic
  html += "<script>\n";
  html += "  const canvas = document.getElementById('chart');\n";
  html += "  const ctx = canvas.getContext('2d');\n";
  html += "  const historyActual = new Array(80).fill(90);\n";
  html += "  const historyTarget = new Array(80).fill(90);\n";
  
  html += "  function drawChart() {\n";
  html += "    ctx.clearRect(0, 0, canvas.width, canvas.height);\n";
  html += "    const step = canvas.width / (historyActual.length - 1);\n";
  
  // Draw Setpoint Target line (dashed red)
  ctx.beginPath();
  ctx.strokeStyle = '#ef4444';
  ctx.lineWidth = 1;
  ctx.setLineDash([4, 4]);
  for (let i = 0; i < historyTarget.length; i++) {
    const y = canvas.height - (historyTarget[i] / 180.0) * canvas.height;
    if (i === 0) ctx.moveTo(0, y); else ctx.lineTo(i * step, y);
  }
  ctx.stroke();
  ctx.setLineDash([]); // Reset
  
  // Draw Actual Position line (solid green)
  ctx.beginPath();
  ctx.strokeStyle = '#10b981';
  ctx.lineWidth = 2.5;
  for (let i = 0; i < historyActual.length; i++) {
    const y = canvas.height - (historyActual[i] / 180.0) * canvas.height;
    if (i === 0) ctx.moveTo(0, y); else ctx.lineTo(i * step, y);
  }
  ctx.stroke();
}

html += "  const ws = new WebSocket('ws://' + window.location.hostname + ':81/');\n";
html += "  ws.onmessage = function(event) {\n";
html += "    const data = JSON.parse(event.data);\n";
html += "    historyActual.push(data.pAct);\n";
html += "    historyActual.shift();\n";
html += "    historyTarget.push(data.pTar);\n";
html += "    historyTarget.shift();\n";
html += "    drawChart();\n";
html += "  };\n";

html += "  function sendTarget() {\n";
html += "    const p = parseFloat(document.getElementById('panIn').value);\n";
html += "    const t = parseFloat(document.getElementById('tiltIn').value);\n";
html += "    document.getElementById('panDisp').innerHTML = p + ' &deg;';\n";
html += "    document.getElementById('tiltDisp').innerHTML = t + ' &deg;';\n";
html += "    ws.send(JSON.stringify({type:'setpoint', pan:p, tilt:t}));\n";
html += "  }\n";

html += "  function sendTuning() {\n";
html += "    const kp = parseFloat(document.getElementById('kpIn').value);\n";
html += "    const kd = parseFloat(document.getElementById('kdIn').value);\n";
html += "    document.getElementById('kpDisp').innerText = kp.toFixed(1);\n";
html += "    document.getElementById('kdDisp').innerText = kd.toFixed(1);\n";
html += "    ws.send(JSON.stringify({type:'tune', kp:kp, kd:kd}));\n";
html += "  }\n";

html += "  window.onload = function() {\n";
html += "    canvas.width = canvas.clientWidth;\n";
    html += "    canvas.height = canvas.clientHeight;\n";
    html += "    drawChart();\n";
    html += "  };\n";
    html += "</script>\n";
    
    html += "</body>\n</html>\n";
    
    server.send(200, "text/html", html);
}

// Parse incoming WebSocket commands (Setpoints + PID Tuning coefficients)
void webSocketEvent(uint8_t num, WSType_t type, uint8_t * payload, size_t length) {
  if (type == WSType_TEXT) {
    StaticJsonDocument<256> doc;
    DeserializationError error = deserializeJson(doc, payload, length);
    
    if (!error && doc.containsKey("type")) {
      String msgType = doc["type"];
      
      if (msgType == "setpoint") {
        panTarget = doc["pan"].as<float>();
        tiltTarget = doc["tilt"].as<float>();
        Serial.printf("[PID Setpoint] Pan target: %.1f | Tilt target: %.1f\n", panTarget, tiltTarget);
      } 
      else if (msgType == "tune") {
        Kp = doc["kp"].as<float>();
        Kd = doc["kd"].as<float>();
        Serial.printf("[PID Tune] Kp: %.2f | Kd: %.2f\n", Kp, Kd);
      }
    }
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Initialize I2C and MPU-6050 Accelerometer
  Wire.begin();
  if (!mpu.begin()) {
    Serial.println("[MPU-6050 Error] Device not found!");
  } else {
    Serial.println("[MPU-6050] Connected successfully.");
  }
  
  // Attach Servos
  panServo.attach(PAN_SERVO_PIN);
  tiltServo.attach(TILT_SERVO_PIN);
  panServo.write(90);
  tiltServo.write(90);
  
  Serial.println("\nESP32 Robotic PID Balancer starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  Serial.print("Local IP Address: ");
  Serial.println(WiFi.localIP());
  
  // Server Routes
  server.on("/", HTTP_GET, handleRoot);
  server.begin();
  
  webSocket.begin();
  webSocket.onEvent(webSocketEvent);
  
  lastTime = millis();
}

void loop() {
  server.handleClient();
  webSocket.loop();
  
  // Calculate delta time
  unsigned long now = millis();
  float dt = (now - lastTime) / 1000.0;
  lastTime = now;
  
  // Execute closed-loop PID calculations (at 50 Hz loop frequency)
  static unsigned long lastPIDTime = 0;
  if (now - lastPIDTime > 20) {
    lastPIDTime = now;
    updateJointPIDs(dt);
  }
  
  // Broadcast joint state telemetries over WebSockets (every 100 ms)
  static unsigned long lastBroadcast = 0;
  if (now - lastBroadcast > 100) {
    lastBroadcast = now;
    
    // Package and send joint states
    String jsonPayload = "{\"pAct\":" + String(panActual, 1) + 
                         ",\"pTar\":" + String(panTarget, 1) + "}";
    webSocket.broadcastTXT(jsonPayload);
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **2 Servos**, **Potentiometer**, and **MPU-6050** onto the canvas.
2. Wire the parts to the specified pins.
3. Paste the code and click **Run**.
4. Open your browser and navigate to the IP address. Verify that the dashboard loads.
5. Move the Pan Target slider on the web page to 130 degrees.
6. In simulation, since joint physical movements are decoupled, simulate joint movement by turning the potentiometer knob on the canvas.
7. Verify that the servo motor widget rotates, matching the potentiometer's physical resistance changes.
8. Adjust the Kp and Kd sliders on the webpage to see the change in system logging.

## Expected Output
Serial Monitor:
```
[MPU-6050] Connected successfully.
WiFi Connected.
Local IP Address: 10.10.0.3
[PID Setpoint] Pan target: 130.0 | Tilt target: 90.0
[PID Tune] Kp: 2.50 | Kd: 0.50
```

## Expected Canvas Behavior
* Modifying target sliders on the webpage sends WebSocket commands, driving the servo motor widgets and updating the canvas tracking charts.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `panTarget - panActual` | Calculates the error value (difference between setpoint target and actual joint position). |
| `Kp * panError + ...` | Standard PID output formula calculating the correction vector. |
| `webSocket.onEvent(...)` | Registers the callback handler to parse text payloads arrived on port 81. |

## Hardware & Safety Concept: Integral Windup and Torque-Limit Brownouts
* **Integral Windup**: If the robotic arm encounters a physical obstacle that prevents it from moving, the PID error stays constant, causing the Integral term (`Ki * error * dt`) to grow very large (windup). When the obstacle is removed, this accumulated value will cause the arm to move violently. Always constrain the maximum integral value:
  `panIntegral = constrain(panIntegral, -100.0, 100.0);`
* **Torque-Limit Brownouts**: High-torque robotic joints can pull several Amps of current when starting or under heavy load. If powered directly from the ESP32's 5V pin, this voltage drop will cause a brownout reset. Always use external power supplies (e.g. 5V 3A AC adapters) for the servo motors.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display "PID Pan/Tilt Active".
2. **Three.js 3D Arm rendering**: Replace the 2D canvas chart with a Three.js 3D viewport (Project 157) rendering a virtual 3D arm mirroring the actual angles.
3. **Integral Tuning**: Add a slider to tune the Integral coefficient (`Ki`) online, saving all tuned PID parameters in NVS.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The joint oscillates wildly | Proportional gain (Kp) too high | If Kp is too high, the joint corrects too violently, overshooting the target and causing high-frequency oscillations. Reduce Kp and increase Kd slightly |
| Servo does not move | Detached servo pin | Servo control signals require hardware PWM channels. Ensure the pins used (GPIO 12, 13) are attached correctly in the library |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [56 - Web Page Servo Controller](../intermediate/56-web-page-servo-angle-controller.md)
- [152 - Web Page Motor PID Speed Graph](152-esp32-web-page-motor-pid-speed-graph.md)
- [198 - Bluetooth & WiFi Robot HUD Console](198-bluetooth-wifi-robot-hud-console.md) (Next project)
