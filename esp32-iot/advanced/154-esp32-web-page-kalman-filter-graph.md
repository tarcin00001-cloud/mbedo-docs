# 154 - Web Page Kalman Filter Graph

Build a real-time signal filtering node on the ESP32 that samples a noisy analog LDR light sensor on GPIO 34, runs a one-dimensional Kalman filter algorithm to smooth the measurement noise, and hosts a web dashboard displaying an HTML5 Canvas graph comparing raw vs. filtered values.

## Goal
Learn how to implement recursive estimation algorithms (1D Kalman Filter), configure process and measurement noise covariance variables, manage circular history buffers, and build dual-line charts.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An LDR is on GPIO 34. The ESP32 samples the analog pin every 100 ms. It processes the raw reading through a Kalman filter algorithm. Navigating to the ESP32's IP address displays a webpage with a live-scrolling line graph plotting both the noisy Raw readings and the smooth Filtered data, along with sliders to tune the filter's responsiveness.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Photoresistor (LDR) | `photoresistor` | Yes | Yes |
| 10 kΩ Resistor (voltage divider) | `resistor` | No | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR Sensor | Pin 1 / Pin 2 | GPIO34 / 3V3 | Yellow / Red | Analog light signal |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 10 kΩ pull-down resistor in series with the LDR to form a voltage divider circuit.

## Code
```cpp
// Web Page Kalman Filter Graph (1D Kalman estimator + Circular buffer + Canvas plot)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int LDR_PIN = 34;

WebServer server(80);

// 1D Kalman Filter variables
double q_processNoise = 0.015;     // Process noise covariance (Q)
double r_measurementNoise = 0.8;  // Measurement noise covariance (R)
double p_errorCovariance = 1.0;    // Estimation error covariance (P)
double x_estimatedValue = 0.0;     // True value estimate (X)
double kalmanGain = 0.0;           // Kalman gain (K)

double rawValue = 0.0;
double filteredValue = 0.0;

unsigned long lastSampleTime = 0;
const unsigned long SAMPLE_INTERVAL_MS = 100; // Sample every 100 ms

// History buffers for plotting (Last 30 data points)
const int HISTORY_SIZE = 30;
float rawHistory[HISTORY_SIZE];
float filteredHistory[HISTORY_SIZE];
int historyIndex = 0;

void addHistoryReading(float raw, float filtered) {
  rawHistory[historyIndex] = raw;
  filteredHistory[historyIndex] = filtered;
  historyIndex = (historyIndex + 1) % HISTORY_SIZE;
}

// 1-Dimensional Kalman Filter update step
double updateKalmanFilter(double measurement) {
  // 1. Prediction update step
  p_errorCovariance = p_errorCovariance + q_processNoise;
  
  // 2. Measurement update step (Calculate Kalman Gain)
  kalmanGain = p_errorCovariance / (p_errorCovariance + r_measurementNoise);
  
  // Update state estimate with measurement correction
  x_estimatedValue = x_estimatedValue + kalmanGain * (measurement - x_estimatedValue);
  
  // Update error covariance
  p_errorCovariance = (1.0 - kalmanGain) * p_errorCovariance;
  
  return x_estimatedValue;
}

// HTTP API endpoint returning JSON data
void handleGetTelemetry() {
  String json = "{\"raw\":" + String(rawValue, 1) + 
                 ",\"filtered\":" + String(filteredValue, 1) + 
                 ",\"q\":" + String(q_processNoise, 4) + 
                 ",\"r\":" + String(r_measurementNoise, 3) + 
                 ",\"raws\":[";
  for (int i = 0; i < HISTORY_SIZE; i++) {
    int idx = (historyIndex + i) % HISTORY_SIZE;
    json += String(rawHistory[idx], 1);
    if (i < HISTORY_SIZE - 1) json += ",";
  }
  json += "],\"filtereds\":[";
  for (int i = 0; i < HISTORY_SIZE; i++) {
    int idx = (historyIndex + i) % HISTORY_SIZE;
    json += String(filteredHistory[idx], 1);
    if (i < HISTORY_SIZE - 1) json += ",";
  }
  json += "]}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to update parameters
void handlePostTune() {
  if (server.hasArg("q")) q_processNoise = server.arg("q").toFloat();
  if (server.hasArg("r")) r_measurementNoise = server.arg("r").toFloat();
  server.send(200, "text/plain", "OK");
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Kalman Filter Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 520px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 20px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 12px; border: 1px solid #334155; text-align: left; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 22px; font-weight: bold; margin-top: 5px; color: #38bdf8; font-family: monospace; }\n";
  html += "  .metric-val.filtered { color: #10b981; }\n";
  html += "  canvas { background-color: #0f172a; border-radius: 8px; border: 1px solid #334155; margin-top: 15px; width: 100%; aspect-ratio: 2/1; }\n";
  html += "  .tuning-box { border-top: 2px solid #334155; padding-top: 15px; margin-top: 20px; text-align: left; }\n";
  html += "  .slider-row { display: flex; align-items: center; gap: 15px; margin-bottom: 12px; }\n";
  html += "  label { font-weight: bold; color: #94a3b8; width: 150px; font-size: 13px; }\n";
  html += "  input[type=range] { flex-grow: 1; height: 8px; border-radius: 4px; background: #334155; outline: none; -webkit-appearance: none; }\n";
  html += "  input[type=range]::-webkit-slider-thumb { -webkit-appearance: none; width: 18px; height: 18px; border-radius: 50%; background: #38bdf8; cursor: pointer; }\n";
  html += "  .value-lbl { font-family: monospace; font-size: 15px; font-weight: bold; width: 60px; text-align: right; color: #38bdf8; }\n";
  html += "  .footer { text-align: center; margin-top: 25px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Kalman Filter Tuner</h1>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Raw Input</div><div class=\"metric-val\" id=\"rawDisplay\">0.0</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Kalman Filtered</div><div class=\"metric-val filtered\" id=\"fltDisplay\">0.0</div></div>\n";
  html += "  </div>\n";
  
  html += "  <canvas id=\"chartCanvas\" width=\"400\" height=\"200\"></canvas>\n";
  
  html += "  <div class=\"tuning-box\">\n";
  html += "    <h3>Covariance Coefficients</h3>\n";
  // Sliders
  html += "    <div class=\"slider-row\"><label>Process Noise (Q)</label><input type=\"range\" id=\"qSldr\" min=\"0.0001\" max=\"0.1\" step=\"0.0005\" value=\"" + String(q_processNoise, 4) + "\" oninput=\"tuneKalman()\"><span id=\"qVal\" class=\"value-lbl\">" + String(q_processNoise, 4) + "</span></div>\n";
  html += "    <div class=\"slider-row\"><label>Measurement Noise (R)</label><input type=\"range\" id=\"rSldr\" min=\"0.01\" max=\"5.0\" step=\"0.05\" value=\"" + String(r_measurementNoise, 2) + "\" oninput=\"tuneKalman()\"><span id=\"rVal\" class=\"value-lbl\">" + String(r_measurementNoise, 2) + "</span></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Blue Line: Raw Data | Green Line: Kalman Estimate</p>\n";
  html += "</div>\n";
  
  // JavaScript dual-line canvas graph
  html += "<script>\n";
  html += "  const canvas = document.getElementById('chartCanvas');\n";
  html += "  const ctx = canvas.getContext('2d');\n";
  
  html += "  function drawChart(raws, filtereds) {\n";
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
  
  html += "    if (raws.length < 2) return;\n";
  const scale = "2.0"; // Scale percentage bounds
  html += "    const step = canvas.width / (raws.length - 1);\n";
  
  // Plot Raw Line (Blue)
  html += "    ctx.strokeStyle = '#38bdf8';\n";
  html += "    ctx.lineWidth = 2.0;\n";
  html += "    ctx.beginPath();\n";
  html += "    raws.forEach((val, i) => {\n";
  html += "      const x = i * step;\n";
  html += "      const y = 200 - (val * " + scale + ");\n";
  html += "      if (i === 0) ctx.moveTo(x, y);\n";
  html += "      else ctx.lineTo(x, y);\n";
  html += "    });\n";
  html += "    ctx.stroke();\n";
  
  // Plot Filtered Line (Green)
  html += "    ctx.strokeStyle = '#10b981';\n";
  html += "    ctx.lineWidth = 3.0;\n";
  html += "    ctx.beginPath();\n";
  html += "    filtereds.forEach((val, i) => {\n";
  html += "      const x = i * step;\n";
  html += "      const y = 200 - (val * " + scale + ");\n";
  html += "      if (i === 0) ctx.moveTo(x, y);\n";
  html += "      else ctx.lineTo(x, y);\n";
  html += "    });\n";
  html += "    ctx.stroke();\n";
  html += "  }\n";
  
  html += "  function updateHUD() {\n";
  html += "    fetch('/api/data')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('rawDisplay').innerText = data.raw.toFixed(1) + '%';\n";
  html += "        document.getElementById('fltDisplay').innerText = data.filtered.toFixed(1) + '%';\n";
  html += "        drawChart(data.raws, data.filtereds);\n";
  // Updated safety indicators
  html += "      });\n";
  html += "  }\n";
  
  html += "  let lastSentTime = 0;\n";
  html += "  function tuneKalman() {\n";
  html += "    const qVal = document.getElementById('qSldr').value;\n";
  html += "    const rVal = document.getElementById('rSldr').value;\n";
  
  html += "    document.getElementById('qVal').innerText = parseFloat(qVal).toFixed(4);\n";
  html += "    document.getElementById('rVal').innerText = parseFloat(rVal).toFixed(2);\n";
  
  html += "    const now = Date.now();\n";
  html += "    if (now - lastSentTime < 100) return;\n";
  html += "    lastSentTime = now;\n";
  
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('q', qVal);\n";
  html += "    body.append('r', rVal);\n";
  
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
  
  pinMode(LDR_PIN, INPUT);
  
  // Fill histories with baseline values
  int baseVal = analogRead(LDR_PIN);
  double mappedBase = (baseVal / 4095.0) * 100.0;
  x_estimatedValue = mappedBase; // Seed initial estimate
  
  for (int i = 0; i < HISTORY_SIZE; i++) {
    rawHistory[i] = mappedBase;
    filteredHistory[i] = mappedBase;
  }
  
  Serial.println("\nESP32 Kalman Filter starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/data", handleGetTelemetry);
  server.on("/api/tune", HTTP_POST, handlePostTune);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  lastSampleTime = millis();
}

void loop() {
  server.handleClient();
  
  // Sample and filter data periodically (every 100 ms)
  unsigned long now = millis();
  if (now - lastSampleTime >= SAMPLE_INTERVAL_MS) {
    lastSampleTime = now;
    
    // 1. Read noisy raw analog input and map to percentage
    int rawRead = analogRead(LDR_PIN);
    
    // Inject artificial noise to clearly demonstrate filter action in simulation
    int noise = random(-300, 301); // Adds +/- 7% noise
    int rawNoisy = constrain(rawRead + noise, 0, 4095);
    
    rawValue = ((double)rawNoisy / 4095.0) * 100.0;
    
    // 2. Filter using 1D Kalman
    filteredValue = updateKalmanFilter(rawValue);
    
    // 3. Append to histories
    addHistoryReading(rawValue, filteredValue);
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and a **Potentiometer** (to simulate the LDR) onto the canvas.
2. Wire the potentiometer wiper to **GPIO34**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. In normal state, verify that the line chart shows a noisy blue line (raw signal) and a smooth green line (Kalman estimate).
6. Tune the sliders on the webpage. Increase the Measurement Noise (R) to see the green line become even smoother but lag slightly behind.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
```

Browser JSON Console (`/api/data`):
```json
{"raw":45.2,"filtered":44.8,"q":0.0150,"r":0.80,"raws":[45.2,46.1,43.8],"filtereds":[44.8,45.1,44.8]}
```

## Expected Canvas Behavior
* Adjusting the simulated sensor potentiometer displays a real-time comparison of raw and filtered lines on the webpage.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `updateKalmanFilter(rawValue)` | Executes the prediction and correction updates of the Kalman Filter. |
| `kalmanGain = ...` | Computes the Kalman Gain, which weights the measurement against the estimate. |
| `q_processNoise` / `r_measurementNoise` | Tunable process and measurement noise variance factors. |

## Hardware & Safety Concept: Sensor Noise Filtering and Dynamic Estimation
* **Sensor Noise Filtering**: Physical sensors are subject to electromagnetic interference, temperature drift, and ADC noise. Simple averaging filters introduce significant delay. A **Kalman Filter** provides optimal smoothing with minimal delay because it dynamically calculates the variance of the measurements.
* **Filter Tuning**: The ratio of Q to R determines the filter behavior. If R (measurement noise) is large, the filter assumes the sensor is noisy and trusts its previous estimate, providing high smoothing but slow response. If Q (process noise) is large, the filter assumes the system is changing rapidly and trusts the sensor, responding fast but showing more noise.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display both raw and filtered values numerically.
2. **Alert sound**: Add a buzzer (GPIO 15) to sound a chime if the estimated light level drops below 20%.
3. **Compare with Simple Moving Average**: Implement a 5-point Simple Moving Average (SMA) in the code and plot it alongside the Kalman filter to compare lag.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The filtered line lags too far behind | R too high or Q too low | Decrease the R slider or increase the Q slider to make the filter react faster |
| The filtered line is still very noisy | Q too high or R too low | Increase the R slider to filter out more noise |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [48 - ESP32 LDR light meter remote report](../beginner/48-esp32-ldr-light-meter-remote-report.md)
- [151 - ESP32 Web Page PID Parameter Tuner](151-esp32-web-page-pid-parameter-tuner.md)
- [155 - Web Page Complementary Filter balancer](155-esp32-web-page-complementary-filter-balancer.md) (Next project)
