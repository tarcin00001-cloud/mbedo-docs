# 151 - Web Page PID Parameter Tuner (Tune coefficients online)

Build a closed-loop control system on the ESP32 that runs a Proportional-Integral-Derivative (PID) algorithm to maintain light levels (reading an LDR on GPIO 34 and adjusting an LED on GPIO 12 using hardware LEDC PWM), and hosts a web dashboard to tune PID coefficients (Kp, Ki, Kd) and setpoints in real time.

## Goal
Learn how to implement mathematical PID algorithms, configure windup guards, configure ESP32 hardware LEDC PWM channels, serve dynamic status pages, and process HTTP POST configuration parameters.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An LED is on GPIO 12, and an LDR on GPIO 34. The ESP32 runs a PID loop every 100 ms to adjust the LED brightness to maintain the light level at the target setpoint. Navigating to the ESP32's IP address displays a dashboard with live inputs, outputs, errors, and sliders to tune Kp, Ki, Kd, and Setpoint values on-the-fly.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Photoresistor (LDR) | `photoresistor` | Yes | Yes |
| 10 kΩ Resistor (voltage divider) | `resistor` | No | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR Sensor | Pin 1 / Pin 2 | GPIO34 / 3V3 | Yellow / Red | Light level signal |
| LED | Anode (+) | GPIO12 | Red | PWM control output |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 220 Ω resistor in series with the LED. Power the LDR voltage divider from the 3.3V rail.

## Code
```cpp
// Web Page PID Parameter Tuner (LDR Light feedback + LED PWM drive + PID algorithm)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int LDR_PIN = 34;
const int LED_PIN = 12;

// LEDC PWM configuration
const int PWM_CHAN = 0;
const int PWM_FREQ = 5000;
const int PWM_RES = 8; // 8-bit resolution (0-255)

WebServer server(80);

// PID Coefficients
double Kp = 2.0;
double Ki = 0.5;
double Kd = 0.1;

// PID Variables
double setpoint = 50.0; // Target light percentage (0-100%)
double input = 0.0;    // Measured light percentage (0-100%)
double output = 0.0;   // PWM output value (0-255)

double error = 0.0;
double lastError = 0.0;
double cumError = 0.0;  // Integral error accumulator
double rateError = 0.0; // Derivative error rate

unsigned long lastPIDTime = 0;
const unsigned long PID_INTERVAL_MS = 100; // Run PID every 100 ms

// Execute PID controller calculations
void computePID() {
  unsigned long now = millis();
  double dt = (double)(now - lastPIDTime) / 1000.0;
  
  if (dt >= 0.1) { // Enforce 100ms execution step
    lastPIDTime = now;
    
    // 1. Read input and convert to percentage (LDR value decreases as light increases)
    int rawVal = analogRead(LDR_PIN);
    input = (1.0 - ((double)rawVal / 4095.0)) * 100.0;
    
    // 2. Calculate error terms
    error = setpoint - input;
    
    // Integral term with anti-windup guard limit
    cumError += error * dt;
    cumError = constrain(cumError, -100.0, 100.0);
    
    // Derivative term
    rateError = (error - lastError) / dt;
    
    // 3. Calculate PID output
    output = (Kp * error) + (Ki * cumError) + (Kd * rateError);
    
    // Constrain output to 8-bit PWM bounds (0-255)
    output = constrain(output, 0.0, 255.0);
    
    // 4. Drive actuator (LED)
    ledcWrite(PWM_CHAN, (int)output);
    
    lastError = error;
  }
}

// HTTP API endpoint returning JSON data
void handleGetStatus() {
  String json = "{\"setpoint\":" + String(setpoint, 1) + 
                 ",\"input\":" + String(input, 1) + 
                 ",\"output\":" + String(output, 1) + 
                 ",\"error\":" + String(error, 1) + 
                 ",\"kp\":" + String(Kp, 2) + 
                 ",\"ki\":" + String(Ki, 2) + 
                 ",\"kd\":" + String(Kd, 2) + "}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to update parameters
void handlePostTune() {
  if (server.hasArg("kp")) Kp = server.arg("kp").toFloat();
  if (server.hasArg("ki")) Ki = server.arg("ki").toFloat();
  if (server.hasArg("kd")) Kd = server.arg("kd").toFloat();
  if (server.hasArg("setpoint")) setpoint = server.arg("setpoint").toFloat();
  
  // Clear accumulator error when parameters change
  cumError = 0;
  
  Serial.printf("[PID Tune] Kp: %.2f | Ki: %.2f | Kd: %.2f | Setpoint: %.1f\n", 
                Kp, Ki, Kd, setpoint);
  server.send(200, "text/plain", "OK");
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>PID Tuner Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 500px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 20px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; text-align: left; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 22px; font-weight: bold; margin-top: 5px; color: #10b981; font-family: monospace; }\n";
  html += "  .tuning-box { border-top: 2px solid #334155; padding-top: 20px; margin-top: 25px; text-align: left; }\n";
  html += "  .slider-row { display: flex; align-items: center; gap: 15px; margin-bottom: 15px; }\n";
  html += "  label { font-weight: bold; color: #94a3b8; width: 80px; font-size: 14px; }\n";
  html += "  input[type=range] { flex-grow: 1; height: 8px; border-radius: 4px; background: #334155; outline: none; -webkit-appearance: none; }\n";
  html += "  input[type=range]::-webkit-slider-thumb { -webkit-appearance: none; width: 18px; height: 18px; border-radius: 50%; background: #38bdf8; cursor: pointer; }\n";
  html += "  .value-lbl { font-family: monospace; font-size: 15px; font-weight: bold; width: 45px; text-align: right; color: #38bdf8; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>PID Parameter Tuner</h1>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Measured Input</div><div class=\"metric-val\" id=\"inputDisplay\">0.0 %</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">PID PWM Output</div><div class=\"metric-val\" id=\"outputDisplay\">0</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Target Setpoint</div><div class=\"metric-val\" id=\"setpointDisplay\">0.0 %</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Active Error</div><div class=\"metric-val\" id=\"errorDisplay\">0.0 %</div></div>\n";
  html += "  </div>\n";
  
  html += "  <div class=\"tuning-box\">\n";
  html += "    <h3>PID Coefficients</h3>\n";
  
  // Kp Slider
  html += "    <div class=\"slider-row\">\n";
  html += "      <label>Proportional (Kp)</label>\n";
  html += "      <input type=\"range\" id=\"kpSldr\" min=\"0\" max=\"10\" step=\"0.1\" value=\"" + String(Kp, 1) + "\" oninput=\"tunePID()\">\n";
  html += "      <span id=\"kpVal\" class=\"value-lbl\">" + String(Kp, 1) + "</span>\n";
  html += "    </div>\n";
  
  // Ki Slider
  html += "    <div class=\"slider-row\">\n";
  html += "      <label>Integral (Ki)</label>\n";
  html += "      <input type=\"range\" id=\"kiSldr\" min=\"0\" max=\"5\" step=\"0.05\" value=\"" + String(Ki, 2) + "\" oninput=\"tunePID()\">\n";
  html += "      <span id=\"kiVal\" class=\"value-lbl\">" + String(Ki, 2) + "</span>\n";
  html += "    </div>\n";
  
  // Kd Slider
  html += "    <div class=\"slider-row\">\n";
  html += "      <label>Derivative (Kd)</label>\n";
  html += "      <input type=\"range\" id=\"kdSldr\" min=\"0\" max=\"2\" step=\"0.02\" value=\"" + String(Kd, 2) + "\" oninput=\"tunePID()\">\n";
  html += "      <span id=\"kdVal\" class=\"value-lbl\">" + String(Kd, 2) + "</span>\n";
  html += "    </div>\n";
  
  // Setpoint Slider
  html += "    <div class=\"slider-row\">\n";
  html += "      <label>Setpoint</label>\n";
  html += "      <input type=\"range\" id=\"setSldr\" min=\"10\" max=\"90\" step=\"1.0\" value=\"" + String(setpoint, 0) + "\" oninput=\"tunePID()\">\n";
  html += "      <span id=\"setVal\" class=\"value-lbl\">" + String(setpoint, 0) + "%</span>\n";
  html += "    </div>\n";
  
  html += "  </div>\n";
  html += "  <p class=\"footer\">PID sample interval: 100 ms | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateHUD() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('inputDisplay').innerText = data.input.toFixed(1) + ' %';\n";
  html += "        document.getElementById('outputDisplay').innerText = Math.round(data.output);\n";
  html += "        document.getElementById('setpointDisplay').innerText = data.setpoint.toFixed(1) + ' %';\n";
  html += "        document.getElementById('errorDisplay').innerText = data.error.toFixed(1) + ' %';\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  let lastSentTime = 0;\n";
  html += "  function tunePID() {\n";
  html += "    const kp = document.getElementById('kpSldr').value;\n";
  html += "    const ki = document.getElementById('kiSldr').value;\n";
  html += "    const kd = document.getElementById('kdSldr').value;\n";
  html += "    const set = document.getElementById('setSldr').value;\n";
  
  html += "    document.getElementById('kpVal').innerText = kp;\n";
  html += "    document.getElementById('kiVal').innerText = ki;\n";
  html += "    document.getElementById('kdVal').innerText = kd;\n";
  html += "    document.getElementById('setVal').innerText = set + '%';\n";
  
  html += "    const now = Date.now();\n";
  html += "    if (now - lastSentTime < 100) return; // Throttle requests\n";
  html += "    lastSentTime = now;\n";
  
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('kp', kp);\n";
  html += "    body.append('ki', ki);\n";
  html += "    body.append('kd', kd);\n";
  html += "    body.append('setpoint', set);\n";
  
  html += "    fetch('/api/tune', { method: 'POST', body: body });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateHUD();\n";
  html += "    setInterval(updateHUD, 200); // Fast update\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LDR_PIN, INPUT);
  
  // Initialize hardware PWM on LEDC
  ledcSetup(PWM_CHAN, PWM_FREQ, PWM_RES);
  ledcAttachPin(LED_PIN, PWM_CHAN);
  ledcWrite(PWM_CHAN, 0);
  
  Serial.println("\nESP32 PID Tuner Node starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/status", handleGetStatus);
  server.on("/api/tune", HTTP_POST, handlePostTune);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  lastPIDTime = millis();
}

void loop() {
  server.handleClient();
  
  // Continuously compute closed loop feedback
  computePID();
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **LED**, and **Potentiometer** (to simulate the LDR) onto the canvas.
2. Wire Potentiometer to **GPIO34** and LED to **GPIO12**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. In normal state, change the target setpoint slider to 60%.
6. Change the simulated LDR potentiometer. Verify that the LED brightness changes to oppose your adjustment (e.g. turning simulated light down makes the LED widget glow brighter).
7. Drag the Kp slider on the webpage. Verify that the response changes speed and stability.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[PID Tune] Kp: 3.5 | Ki: 0.5 | Kd: 0.1 | Setpoint: 60.0
```

## Expected Canvas Behavior
* Adjusting the simulated LDR potentiometer dynamically varies the brightness of the LED widget via closed-loop PID control.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `ledcSetup(...)` / `ledcAttachPin(...)` | Sets up the ESP32 hardware PWM peripheral. |
| `setpoint - input` | Calculates the error (difference between target and measured values). |
| `cumError += error * dt` | Accumulates integral error to eliminate steady-state offset. |
| `output = (Kp * error) + ...` | Sums Proportional, Integral, and Derivative terms to calculate correction output. |

## Hardware & Safety Concept: Integral Windup and Actuator Saturation
* **Integral Windup**: If the LED is already at maximum brightness (`255`) but the target setpoint is still not reached, the error remains positive. The integral term (`cumError`) will continue to grow larger and larger. When the setpoint is finally reached, this accumulated error will cause the LED to stay at maximum brightness for too long. Implementing an **anti-windup guard** (using `constrain` to limit `cumError` to `-100` to `100`) protects the control loop from overshoot.
* **Actuator Saturation**: Always constrain the final calculation output to the physical limits of the actuator (e.g., 0 to 255 for 8-bit PWM) before driving the output pin.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a real-time line graph of the measured light value.
2. **Error alarm buzzer**: Add a buzzer (GPIO 15) to sound an alarm if the error remains high (> 20%) for more than 5 seconds.
3. **Temp PID**: Adapt the code to control a heater relay based on DHT22 temperature feedback.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The LED oscillates wildly | Proportional gain (Kp) too high | Decrease the Kp slider value on the webpage to stabilize the loop |
| The LED takes too long to adjust | Integral gain (Ki) too low | Increase the Ki slider value slightly to eliminate steady-state error |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [49 - Potentiometer LED Bar Graph level controller](../beginner/49-potentiometer-led-bar-graph-level-controller.md)
- [150 - ESP32 Dual-protocol WiFi-to-SD logger](150-esp32-dual-protocol-wifi-to-sd-logger.md)
- [152 - Web Page Motor PID Speed Graph](152-esp32-web-page-motor-pid-speed-graph.md) (Next project)
