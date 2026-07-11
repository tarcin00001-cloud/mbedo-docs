# 152 - Web Page Motor PID Speed Graph

Build a closed-loop DC motor speed controller on the ESP32 that measures motor speed using a rotary encoder input on GPIO 34, runs a PID algorithm to adjust PWM output via an L298N motor driver (on GPIOs 27/26), and hosts a web dashboard with a live-scrolling HTML5 Canvas chart plotting Target vs. Actual RPM.

## Goal
Learn how to configure external interrupts for rotary encoder pulse counting, calculate rotational speed (RPM), implement PID speed control loops, and build real-time multi-line charts on HTML5 Canvas.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An L298N driver is on GPIOs 27/26, and an encoder sensor on GPIO 34. The ESP32 calculates motor speed (RPM) using interrupts. A PID loop runs every 100 ms to adjust the PWM output to match a target RPM setpoint. Navigating to the ESP32's IP address displays a webpage plotting both Target RPM and Actual RPM trends in real time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N H-Bridge Motor Driver | `l298n` | Yes | Yes |
| DC Gear Motor with Encoder | `motor_dc` | Yes | Yes |
| Shared power / Battery Pack | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use a potentiometer to simulate the encoder feedback frequency, or read simulated motor speed directly from the motor widget.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DC Motor Encoder | OUT (Signal A) | GPIO34 | Yellow | Speed pulse input |
| L298N Driver | IN1 / IN2 | GPIO27 / GPIO26 | Blue / White | Motor direction & PWM |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the L298N driver from an external 5V/6V battery pack and connect its ground to the ESP32 ground pin.

## Code
```cpp
// Web Page Motor PID Speed Graph (DC Motor Speed control + Interrupt counter + Canvas chart)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// L298N pins
const int IN1_PWM_PIN = 27; // Configured for PWM speed control
const int IN2_DIR_PIN = 26; // Direction control pin

// Encoder pin
const int ENCODER_PIN = 34;

// LEDC PWM Configuration
const int PWM_CHAN = 0;
const int PWM_FREQ = 10000;
const int PWM_RES = 8; // 8-bit (0-255)

WebServer server(80);

// Encoder counter variables
volatile int pulseCount = 0;
const float PULSES_PER_REV = 20.0; // Pulses per revolution of encoder disc

// PID parameters
double Kp = 1.8;
double Ki = 0.6;
double Kd = 0.05;

double targetRPM = 120.0;
double currentRPM = 0.0;
double pwmOutput = 0.0;

double error = 0.0;
double lastError = 0.0;
double cumError = 0.0;
double rateError = 0.0;

unsigned long lastPIDTime = 0;
const unsigned long PID_INTERVAL_MS = 100; // Run PID every 100 ms

// History buffers for plotting (Last 30 data points)
const int HISTORY_SIZE = 30;
float targetHistory[HISTORY_SIZE];
float actualHistory[HISTORY_SIZE];
int historyIndex = 0;

// Interrupt Service Routine (ISR) for encoder pulse counting
void IRAM_ATTR encoderISR() {
  pulseCount++;
}

void addHistoryReading(float target, float actual) {
  targetHistory[historyIndex] = target;
  actualHistory[historyIndex] = actual;
  historyIndex = (historyIndex + 1) % HISTORY_SIZE;
}

// Compute motor speed and execute PID correction loop
void computeMotorPID() {
  unsigned long now = millis();
  double dt = (double)(now - lastPIDTime) / 1000.0;
  
  if (dt >= 0.1) { // 100 ms interval check
    lastPIDTime = now;
    
    // 1. Calculate current speed in RPM
    // RPM = (pulses / PPR) * (60 / dt)
    noInterrupts(); // Temporarily disable interrupts to copy volatile variable safely
    int pulses = pulseCount;
    pulseCount = 0;
    interrupts();
    
    currentRPM = ((float)pulses / PULSES_PER_REV) * (60.0 / dt);
    
    // Fallback simulation in MbedO if motor isn't spinning physically
    if (pwmOutput > 10.0 && currentRPM == 0.0) {
      // Simulate motor response with lag: speed tracks PWM output
      currentRPM = currentRPM * 0.7 + (pwmOutput * 0.8) * 0.3;
    }
    
    addHistoryReading(targetRPM, currentRPM);
    
    // 2. PID Algorithm
    error = targetRPM - currentRPM;
    
    // Integral with anti-windup constraint
    cumError += error * dt;
    cumError = constrain(cumError, -150.0, 150.0);
    
    // Derivative
    rateError = (error - lastError) / dt;
    
    // Sum terms
    pwmOutput = (Kp * error) + (Ki * cumError) + (Kd * rateError);
    pwmOutput = constrain(pwmOutput, 0.0, 255.0);
    
    // 3. Drive L298N
    digitalWrite(IN2_DIR_PIN, LOW); // Set forward direction
    ledcWrite(PWM_CHAN, (int)pwmOutput);
    
    lastError = error;
  }
}

// HTTP API endpoint returning JSON data
void handleGetTelemetry() {
  String json = "{\"target\":" + String(targetRPM, 1) + 
                 ",\"actual\":" + String(currentRPM, 1) + 
                 ",\"pwm\":" + String(pwmOutput, 1) + 
                 ",\"targets\":[";
  for (int i = 0; i < HISTORY_SIZE; i++) {
    int idx = (historyIndex + i) % HISTORY_SIZE;
    json += String(targetHistory[idx], 1);
    if (i < HISTORY_SIZE - 1) json += ",";
  }
  json += "],\"actuals\":[";
  for (int i = 0; i < HISTORY_SIZE; i++) {
    int idx = (historyIndex + i) % HISTORY_SIZE;
    json += String(actualHistory[idx], 1);
    if (i < HISTORY_SIZE - 1) json += ",";
  }
  json += "]}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to update parameters
void handlePostTune() {
  if (server.hasArg("kp")) Kp = server.arg("kp").toFloat();
  if (server.hasArg("ki")) Ki = server.arg("ki").toFloat();
  if (server.hasArg("kd")) Kd = server.arg("kd").toFloat();
  if (server.hasArg("target")) targetRPM = server.arg("target").toFloat();
  
  cumError = 0; // Reset integral term on tune
  server.send(200, "text/plain", "OK");
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Motor PID Tuner</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 520px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 20px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 12px; border: 1px solid #334155; text-align: left; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 22px; font-weight: bold; margin-top: 5px; color: #10b981; font-family: monospace; }\n";
  html += "  canvas { background-color: #0f172a; border-radius: 8px; border: 1px solid #334155; margin-top: 15px; width: 100%; aspect-ratio: 2/1; }\n";
  html += "  .tuning-box { border-top: 2px solid #334155; padding-top: 15px; margin-top: 20px; text-align: left; }\n";
  html += "  .slider-row { display: flex; align-items: center; gap: 15px; margin-bottom: 12px; }\n";
  html += "  label { font-weight: bold; color: #94a3b8; width: 90px; font-size: 13px; }\n";
  html += "  input[type=range] { flex-grow: 1; height: 8px; border-radius: 4px; background: #334155; outline: none; -webkit-appearance: none; }\n";
  html += "  input[type=range]::-webkit-slider-thumb { -webkit-appearance: none; width: 18px; height: 18px; border-radius: 50%; background: #38bdf8; cursor: pointer; }\n";
  html += "  .value-lbl { font-family: monospace; font-size: 15px; font-weight: bold; width: 45px; text-align: right; color: #38bdf8; }\n";
  html += "  .footer { text-align: center; margin-top: 25px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Motor Speed PID Tuner</h1>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Target RPM</div><div class=\"metric-val\" id=\"tgtDisplay\" style=\"color:#ef4444;\">0.0</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Actual RPM</div><div class=\"metric-val\" id=\"actDisplay\">0.0</div></div>\n";
  html += "  </div>\n";
  
  html += "  <canvas id=\"chartCanvas\" width=\"400\" height=\"200\"></canvas>\n";
  
  html += "  <div class=\"tuning-box\">\n";
  // Sliders
  html += "    <div class=\"slider-row\"><label>Prop (Kp)</label><input type=\"range\" id=\"kpSldr\" min=\"0\" max=\"5\" step=\"0.1\" value=\"" + String(Kp, 1) + "\" oninput=\"tunePID()\"><span id=\"kpVal\" class=\"value-lbl\">" + String(Kp, 1) + "</span></div>\n";
  html += "    <div class=\"slider-row\"><label>Int (Ki)</label><input type=\"range\" id=\"kiSldr\" min=\"0\" max=\"3\" step=\"0.05\" value=\"" + String(Ki, 2) + "\" oninput=\"tunePID()\"><span id=\"kiVal\" class=\"value-lbl\">" + String(Ki, 2) + "</span></div>\n";
  html += "    <div class=\"slider-row\"><label>Deriv (Kd)</label><input type=\"range\" id=\"kdSldr\" min=\"0\" max=\"1\" step=\"0.01\" value=\"" + String(Kd, 2) + "\" oninput=\"tunePID()\"><span id=\"kdVal\" class=\"value-lbl\">" + String(Kd, 2) + "</span></div>\n";
  html += "    <div class=\"slider-row\"><label>Target RPM</label><input type=\"range\" id=\"setSldr\" min=\"30\" max=\"200\" step=\"5\" value=\"" + String(targetRPM, 0) + "\" oninput=\"tunePID()\"><span id=\"setVal\" class=\"value-lbl\">" + String(targetRPM, 0) + "</span></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Polling: 200 ms | Encoder pulses per rev: 20</p>\n";
  html += "</div>\n";
  
  // JavaScript multi-line canvas graph
  html += "<script>\n";
  html += "  const canvas = document.getElementById('chartCanvas');\n";
  html += "  const ctx = canvas.getContext('2d');\n";
  
  html += "  function drawChart(targets, actuals) {\n";
  html += "    ctx.clearRect(0, 0, canvas.width, canvas.height);\n";
  
  // Grid Lines
  html += "    ctx.strokeStyle = '#334155';\n";
  html += "    ctx.lineWidth = 1;\n";
  html += "    for(let y = 50; y < 200; y += 50) {\n";
  html += "      ctx.beginPath();\n";
  html += "      ctx.moveTo(0, y);\n";
  html += "      ctx.lineTo(canvas.width, y);\n";
  html += "      ctx.stroke();\n";
  html += "    }\n";
  
  html += "    if (targets.length < 2) return;\n";
  html += "    const step = canvas.width / (targets.length - 1);\n";
  
  // Plot Target Line (Red)
  html += "    ctx.strokeStyle = '#ef4444';\n";
  html += "    ctx.lineWidth = 2.5;\n";
  html += "    ctx.beginPath();\n";
  html += "    targets.forEach((val, i) => {\n";
  html += "      const x = i * step;\n";
  html += "      const y = 200 - (val * 0.8);\n";
  html += "      if (i === 0) ctx.moveTo(x, y);\n";
  html += "      else ctx.lineTo(x, y);\n";
  html += "    });\n";
  html += "    ctx.stroke();\n";
  
  // Plot Actual Line (Green)
  html += "    ctx.strokeStyle = '#10b981';\n";
  html += "    ctx.lineWidth = 2.5;\n";
  html += "    ctx.beginPath();\n";
  html += "    actuals.forEach((val, i) => {\n";
  html += "      const x = i * step;\n";
  html += "      const y = 200 - (val * 0.8);\n";
  html += "      if (i === 0) ctx.moveTo(x, y);\n";
  html += "      else ctx.lineTo(x, y);\n";
  html += "    });\n";
  html += "    ctx.stroke();\n";
  html += "  }\n";
  
  html += "  function updateHUD() {\n";
  html += "    fetch('/api/telemetry')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('tgtDisplay').innerText = data.target.toFixed(1);\n";
  html += "        document.getElementById('actDisplay').innerText = data.actual.toFixed(1);\n";
  html += "        drawChart(data.targets, data.actuals);\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  let lastSentTime = 0;\n";
  html += "  function tunePID() {\n";
  html += "    const kp = document.getElementById('kpSldr').value;\n";
  html += "    const ki = document.getElementById('kiSldr').value;\n";
  html += "    const kd = document.getElementById('kdSldr').value;\n";
  html += "    const target = document.getElementById('setSldr').value;\n";
  
  html += "    document.getElementById('kpVal').innerText = kp;\n";
  html += "    document.getElementById('kiVal').innerText = ki;\n";
  html += "    document.getElementById('kdVal').innerText = kd;\n";
  html += "    document.getElementById('setVal').innerText = target;\n";
  
  html += "    const now = Date.now();\n";
  html += "    if (now - lastSentTime < 100) return;\n";
  html += "    lastSentTime = now;\n";
  
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('kp', kp);\n";
  html += "    body.append('ki', ki);\n";
  html += "    body.append('kd', kd);\n";
  html += "    body.append('target', target);\n";
  
  html += "    fetch('/api/tune', { method: 'POST', body: body });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateHUD();\n";
  html += "    setInterval(updateHUD, 200);\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(ENCODER_PIN, INPUT_PULLUP);
  
  // Attach external interrupt to count encoder pulses on RISING edge
  attachInterrupt(digitalPinToInterrupt(ENCODER_PIN), encoderISR, RISING);
  
  // Configure L298N pins
  pinMode(IN2_DIR_PIN, OUTPUT);
  ledcSetup(PWM_CHAN, PWM_FREQ, PWM_RES);
  ledcAttachPin(IN1_PWM_PIN, PWM_CHAN);
  
  // Initialize histories with zeros
  for (int i = 0; i < HISTORY_SIZE; i++) {
    targetHistory[i] = 0.0;
    actualHistory[i] = 0.0;
  }
  
  Serial.println("\nESP32 Speed PID Node starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/telemetry", handleGetTelemetry);
  server.on("/api/tune", HTTP_POST, handlePostTune);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  lastPIDTime = millis();
}

void loop() {
  server.handleClient();
  
  // Run motor control calculations
  computeMotorPID();
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N Driver**, **DC Motor**, and a **Potentiometer** (to simulate the encoder pulse rate on GPIO34) onto the canvas.
2. Wire IN1/IN2 to **GPIO27/GPIO26**, and Potentiometer wiper to **GPIO34**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Set the target RPM to 120 on the webpage.
6. Slowly slide the simulated potentiometer. Verify that the webpage line graph draws two lines, showing the actual speed converging on the target line.
7. Change the Kp and Ki values to see how the motor responds to tuning.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
```

Browser JSON Console (`/api/telemetry`):
```json
{"target":120.0,"actual":118.5,"pwm":154.2,"targets":[120,120,120],"actuals":[115,117,118.5]}
```

## Expected Canvas Behavior
* Adjusting the simulated encoder speed potentiometer shows the target vs. actual motor speed curves on the webpage.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `attachInterrupt(...)` | Registers the encoder pulse counting interrupt service routine. |
| `noInterrupts()` | Disables interrupts temporarily to read volatile counters without race conditions. |
| `targetRPM - currentRPM` | Computes the speed error term. |
| `ledcWrite(PWM_CHAN, ...)` | Sets the L298N driver PWM speed. |

## Hardware & Safety Concept: Interrupt Safety and Back-EMF Suppression
* **Interrupt Safety**: Code inside Interrupt Service Routines (ISRs) must execute as fast as possible (under 1–2 µs). Never use `delay()`, `Serial.print()`, or mathematical divisions inside an ISR. Declare any shared variables as `volatile` to prevent compilers from optimizing them incorrectly, and disable interrupts briefly in the main code when reading them.
* **Back-EMF Suppression**: DC motors generate large inductive voltage spikes (Back-EMF) when starting, stopping, or reversing. If not suppressed, these spikes can destroy the motor driver or the ESP32. Ensure that you place flyback diodes (like 1N4007) across the motor terminals, and solder 100 nF capacitors between the motor terminals and chassis to absorb high-frequency electrical noise.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the current speed numerically.
2. **Speed overload cut-off**: Add a buzzer (GPIO 15) to sound an alarm and stop the motor if the RPM exceeds 220.
3. **SPIFFS integration**: Log RPM history to a CSV file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| RPM reads zero constantly | Interrupt pin wrong | Check that the encoder signal is wired to the correct GPIO pin (GPIO 34) and is set to `INPUT_PULLUP` |
| Motor runs at full speed | IN1/IN2 wiring wrong | Check your L298N direction pin logic. Make sure IN2 is set to LOW for forward direction |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [121 - ESP32 WiFi Web Page Drive Controller](121-esp32-wifi-web-page-drive-controller.md)
- [151 - ESP32 Web Page PID Parameter Tuner](151-esp32-web-page-pid-parameter-tuner.md)
- [153 - Web-controlled Temperature PID Heater Node](153-web-controlled-temperature-pid-heater-node.md) (Next project)
