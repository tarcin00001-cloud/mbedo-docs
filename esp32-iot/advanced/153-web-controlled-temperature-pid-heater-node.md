# 153 - Web-controlled Temperature PID Heater Node

Build a closed-loop thermal control system on the ESP32 that samples a DHT22 temperature sensor on GPIO 14, runs a PID time-proportioning control algorithm to actuate a heating relay on GPIO 12, and hosts a web dashboard to adjust coefficients and target setpoints.

## Goal
Learn how to implement slow-switching time-proportioning PWM loops (necessary for mechanical relays and solid-state relays), calculate thermal PID error terms, serve web dashboards, and parse HTTP POST settings.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A DHT22 sensor is on GPIO 14, and a relay module on GPIO 12. The ESP32 calculates temperature. Every 2 seconds, it runs a PID loop. Because a relay cannot switch at high frequencies, the code translates the PID output (0–100% duty cycle) into a 2-second time window (e.g. 40% duty cycle = relay ON for 0.8s, OFF for 1.2s). A webpage displays live values and tuning sliders.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temperature & Humidity Sensor | `dht22` | Yes | Yes |
| Relay Module (SSR or Mechanical) | `relay` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | OUT (Signal) | GPIO14 | Yellow | Climate sensor signal |
| Relay Module | IN (Signal) | GPIO12 | Blue | Heater control signal |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the DHT22 from the 3.3V rail and the relay from the 5V Vin rail.

## Code
```cpp
// Web-controlled Temperature PID Heater Node (DHT22 sensor + Time-Proportioning Relay + PID)
#include <WiFi.h>
#include <WebServer.h>
#include <DHT.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int DHT_PIN = 14;
#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

const int RELAY_PIN = 12;

WebServer server(80);

// PID Coefficients
double Kp = 12.5;
double Ki = 0.8;
double Kd = 5.0;

// PID Variables
double setpointTemp = 28.0; // Target temperature in Celsius
double inputTemp = 24.0;    // Measured temperature
double outputDuty = 0.0;    // Output duty cycle percentage (0-100%)

double error = 0.0;
double lastError = 0.0;
double cumError = 0.0;
double rateError = 0.0;

unsigned long lastPIDTime = 0;
const unsigned long PID_INTERVAL_MS = 2000; // Run PID loop every 2 seconds (matching DHT22 limits)

// Time-Proportioning parameters (Slow PWM for relays)
unsigned long windowStartTime = 0;
const unsigned long WINDOW_SIZE_MS = 2000; // 2-second time window

// Compute thermal PID feedback and update slow-switching relay duty cycle
void computeThermalPID() {
  unsigned long now = millis();
  double dt = (double)(now - lastPIDTime) / 1000.0;
  
  if (dt >= 2.0) { // Run PID calculations every 2 seconds
    lastPIDTime = now;
    
    // 1. Read temperature
    float t = dht.readTemperature();
    if (!isnan(t)) {
      inputTemp = t;
    }
    
    // 2. Calculate error
    error = setpointTemp - inputTemp;
    
    // Integral with anti-windup guard limit
    cumError += error * dt;
    cumError = constrain(cumError, -50.0, 50.0);
    
    // Derivative
    rateError = (error - lastError) / dt;
    
    // Sum PID terms (output represents heating duty cycle percentage: 0 to 100)
    outputDuty = (Kp * error) + (Ki * cumError) + (Kd * rateError);
    outputDuty = constrain(outputDuty, 0.0, 100.0);
    
    lastError = error;
    
    Serial.printf("[PID] Temp: %.1f C | Target: %.1f C | Output Duty: %.1f%%\n", 
                  inputTemp, setpointTemp, outputDuty);
  }
  
  // 3. Time-Proportioning Actuator Drive (Slow PWM)
  if (now - windowStartTime >= WINDOW_SIZE_MS) {
    windowStartTime = now; // Reset time window
  }
  
  // Determine if relay should be closed (ON) or open (OFF) for this slice of the window
  double activeDutyMs = (outputDuty / 100.0) * WINDOW_SIZE_MS;
  if ((now - windowStartTime) < activeDutyMs) {
    digitalWrite(RELAY_PIN, HIGH); // Heat ON
  } else {
    digitalWrite(RELAY_PIN, LOW);  // Heat OFF
  }
}

// HTTP API endpoint returning JSON data
void handleGetStatus() {
  String json = "{\"setpoint\":" + String(setpointTemp, 1) + 
                 ",\"temp\":" + String(inputTemp, 1) + 
                 ",\"duty\":" + String(outputDuty, 1) + 
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
  if (server.hasArg("setpoint")) setpointTemp = server.arg("setpoint").toFloat();
  
  cumError = 0; // Reset accumulator on retune
  server.send(200, "text/plain", "OK");
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Heater PID Console</title>\n";
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
  html += "  label { font-weight: bold; color: #94a3b8; width: 90px; font-size: 13px; }\n";
  html += "  input[type=range] { flex-grow: 1; height: 8px; border-radius: 4px; background: #334155; outline: none; -webkit-appearance: none; }\n";
  html += "  input[type=range]::-webkit-slider-thumb { -webkit-appearance: none; width: 18px; height: 18px; border-radius: 50%; background: #38bdf8; cursor: pointer; }\n";
  html += "  .value-lbl { font-family: monospace; font-size: 15px; font-weight: bold; width: 55px; text-align: right; color: #38bdf8; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Heater PID Controller</h1>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Measured Temp</div><div class=\"metric-val\" id=\"tempDisplay\">0.0 &deg;C</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Heater Duty</div><div class=\"metric-val\" id=\"dutyDisplay\">0.0 %</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Target Setpoint</div><div class=\"metric-val\" id=\"setpointDisplay\">0.0 &deg;C</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Active Error</div><div class=\"metric-val\" id=\"errorDisplay\">0.0 &deg;C</div></div>\n";
  html += "  </div>\n";
  
  html += "  <div class=\"tuning-box\">\n";
  html += "    <h3>PID Calibration</h3>\n";
  
  // Sliders
  html += "    <div class=\"slider-row\"><label>Prop (Kp)</label><input type=\"range\" id=\"kpSldr\" min=\"0\" max=\"30\" step=\"0.5\" value=\"" + String(Kp, 1) + "\" oninput=\"tunePID()\"><span id=\"kpVal\" class=\"value-lbl\">" + String(Kp, 1) + "</span></div>\n";
  html += "    <div class=\"slider-row\"><label>Int (Ki)</label><input type=\"range\" id=\"kiSldr\" min=\"0\" max=\"5\" step=\"0.05\" value=\"" + String(Ki, 2) + "\" oninput=\"tunePID()\"><span id=\"kiVal\" class=\"value-lbl\">" + String(Ki, 2) + "</span></div>\n";
  html += "    <div class=\"slider-row\"><label>Deriv (Kd)</label><input type=\"range\" id=\"kdSldr\" min=\"0\" max=\"15\" step=\"0.2\" value=\"" + String(Kd, 1) + "\" oninput=\"tunePID()\"><span id=\"kdVal\" class=\"value-lbl\">" + String(Kd, 1) + "</span></div>\n";
  html += "    <div class=\"slider-row\"><label>Target Temp</label><input type=\"range\" id=\"setSldr\" min=\"15\" max=\"45\" step=\"0.5\" value=\"" + String(setpointTemp, 1) + "\" oninput=\"tunePID()\"><span id=\"setVal\" class=\"value-lbl\">" + String(setpointTemp, 1) + "&deg;C</span></div>\n";
  
  html += "  </div>\n";
  html += "  <p class=\"footer\">Heater relay: CS pin 12 | PID period: 2 seconds</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateHUD() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('tempDisplay').innerHTML = data.temp.toFixed(1) + ' &deg;C';\n";
  html += "        document.getElementById('dutyDisplay').innerText = data.duty.toFixed(1) + ' %';\n";
  html += "        document.getElementById('setpointDisplay').innerHTML = data.setpoint.toFixed(1) + ' &deg;C';\n";
  html += "        document.getElementById('errorDisplay').innerHTML = data.error.toFixed(1) + ' &deg;C';\n";
  // Updated safety checks
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
  html += "    document.getElementById('setVal').innerHTML = set + '&deg;C';\n";
  
  html += "    const now = Date.now();\n";
  html += "    if (now - lastSentTime < 100) return;\n";
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
  html += "    setInterval(updateHUD, 1000); // Polling index\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  dht.begin();
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW);
  
  Serial.println("\nESP32 Heater PID starting...");
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
  windowStartTime = millis();
}

void loop() {
  server.handleClient();
  
  // Compute thermal loop
  computeThermalPID();
  
  delay(5);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22 Sensor**, and **Relay** onto the canvas.
2. Wire DHT22 output to **GPIO14** and Relay input to **GPIO12**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Set the target setpoint to 30.0 °C.
6. Set the simulated DHT22 temperature to 24.0 °C.
7. Verify that the relay widget closes (ON) and then opens (OFF) in a slow cycle, and the console outputs the duty cycle calculations.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[PID] Temp: 24.0 C | Target: 30.0 C | Output Duty: 75.0%
```

## Expected Canvas Behavior
* The simulated relay clicks open and closed based on calculated heating duty cycle ratios.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `dht.readTemperature()` | Samples the current ambient temperature. |
| `outputDuty = (Kp * error) + ...` | Computes the duty cycle (0 to 100%) needed to reach setpoint. |
| `activeDutyMs = (outputDuty / 100.0) * WINDOW_SIZE_MS` | Converts duty cycle percentage to milliseconds of ON-time. |
| `digitalWrite(RELAY_PIN, HIGH)` | Closes the relay contacts to activate the heating element. |

## Hardware & Safety Concept: Slow PWM and Thermal Runaway Protection
* **Slow PWM**: Standard high-frequency PWM (e.g. 5000 Hz) will quickly destroy mechanical relays due to electric arcing and contact wear. Solid-state relays can switch faster, but time-proportioning control with a **slow period** (e.g. 2 seconds) is the standard approach for managing heavy heating elements.
* **Thermal Runaway**: If the temperature sensor fails (returning `NaN` or a frozen low reading), the PID loop will stay at 100% duty cycle, creating a fire hazard. Always implement a safety timeout: if the DHT22 returns invalid readings for more than 3 consecutive samples, shut down the relay immediately.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a flame icon if the heater is active.
2. **Alert sound**: Add a buzzer (GPIO 15) to sound an alarm if the temperature exceeds 45.0 °C (overheat safety).
3. **SPIFFS integration**: Save thermal history logs to a CSV file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay clicks continuously and rapidly | Window size too short | Ensure the `WINDOW_SIZE_MS` is set to at least 2000 ms to prevent rapid cycling |
| Temperature never reaches setpoint | Heater power too low | Increase the Kp slider value to boost heating drive during high error states |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [105 - ESP32 Thermostat Controller with Web Adjustments](105-esp32-thermostat-controller-iot-ntc-relay-lcd-web-threshold-adjust.md)
- [151 - ESP32 Web Page PID Parameter Tuner](151-esp32-web-page-pid-parameter-tuner.md)
- [154 - Web Page Kalman Filter Graph](154-esp32-web-page-kalman-filter-graph.md) (Next project)
