# 155 - Web Page Complementary Filter balancer

Build a real-time orientation estimator on the ESP32 that interfaces with an MPU-6050 6-axis IMU over I2C (GPIO 21/22), runs a Complementary Filter algorithm to combine accelerometer pitch angles and gyroscope angular velocity rates, and hosts a web dashboard displaying an HTML5 Canvas line chart of the raw vs. filtered angles.

## Goal
Learn how to interface IMU sensors over I2C, perform trigonometric angle calculations, implement Complementary Filter math, manage circular history buffers, and build multi-line graphs.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An MPU-6050 sensor is on I2C (GPIO 21/22). Every 50 ms, the ESP32 samples linear accelerations and angular rates. It computes pitch using the accelerometer, integrates the gyroscope, and merges both using a Complementary Filter (`angle = alpha * (angle + gyro * dt) + (1-alpha) * accel`). Navigating to the ESP32's IP address displays a webpage plotting Accel, Gyro, and Filtered trends.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MPU-6050 6-Axis IMU Sensor | `i2c_device` | No | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use two potentiometers (wired to GPIOs 34 and 35) to simulate the raw accelerometer and gyroscope pitch rates if the physical MPU-6050 is not present.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 Sensor | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the MPU-6050 sensor from the 3.3V pin. Ensure pull-up resistors are enabled on SDA/SCL lines.

## Code
```cpp
// Web Page Complementary Filter balancer (MPU-6050 I2C interface + Complementary math + Web graph)
#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

Adafruit_MPU6050 mpu;
WebServer server(80);

// Complementary filter variables
double alpha = 0.96; // Filter coefficient weighting gyroscope vs accelerometer
double pitchAccel = 0.0;
double pitchGyro = 0.0;
double pitchFiltered = 0.0;

unsigned long lastFilterTime = 0;
const unsigned long FILTER_INTERVAL_MS = 50; // Filter update every 50 ms

// History buffers for plotting (Last 30 data points)
const int HISTORY_SIZE = 30;
float accelHistory[HISTORY_SIZE];
float gyroHistory[HISTORY_SIZE];
float filteredHistory[HISTORY_SIZE];
int historyIndex = 0;

void addHistoryReading(float acc, float gyr, float flt) {
  accelHistory[historyIndex] = acc;
  gyroHistory[historyIndex] = gyr;
  filteredHistory[historyIndex] = flt;
  historyIndex = (historyIndex + 1) % HISTORY_SIZE;
}

// Read IMU and execute complementary filter calculations
void computeComplementaryFilter() {
  unsigned long now = millis();
  double dt = (double)(now - lastFilterTime) / 1000.0;
  
  if (dt >= 0.05) { // 50 ms step
    lastFilterTime = now;
    
    double accelX = 0.0, accelY = 0.0, accelZ = 9.8;
    double gyroY = 0.0; // Pitch rate (deg/sec)
    
    sensors_event_t a, g, temp;
    if (mpu.getEvent(&a, &g, &temp)) {
      accelX = a.acceleration.x;
      accelY = a.acceleration.y;
      accelZ = a.acceleration.z;
      
      // Convert gyro rate from rad/sec to deg/sec
      gyroY = g.gyro.y * 180.0 / PI;
    } else {
      // Simulation fallback values if IMU is missing
      static double angleMock = 0.0;
      angleMock += ((double)random(-10, 11) / 10.0); // Drift simulation
      accelX = sin(angleMock * PI / 180.0) * 9.8;
      accelZ = cos(angleMock * PI / 180.0) * 9.8;
      gyroY = (double)random(-5, 6) / 5.0; // Random angular rates
    }
    
    // 1. Compute Pitch from Accelerometer data
    // pitchAccel = atan2(-accelX, sqrt(accelY*accelY + accelZ*accelZ)) * 180 / PI
    pitchAccel = atan2(-accelX, sqrt(accelY * accelY + accelZ * accelZ)) * 180.0 / PI;
    
    // 2. Compute Pitch from Gyroscope integration (Notice: drifts over time)
    pitchGyro = pitchGyro + (gyroY * dt);
    
    // 3. Apply Complementary Filter
    // angle = alpha * (angle + gyro_rate * dt) + (1-alpha) * accel_angle
    pitchFiltered = alpha * (pitchFiltered + gyroY * dt) + (1.0 - alpha) * pitchAccel;
    
    // Keep gyro reading in bounds for plotting comparison
    if (abs(pitchGyro) > 90.0) pitchGyro = pitchAccel; // Reset gyro drift when extreme
    
    addHistoryReading(pitchAccel, pitchGyro, pitchFiltered);
  }
}

// HTTP API endpoint returning JSON data
void handleGetTelemetry() {
  String json = "{\"accel\":" + String(pitchAccel, 1) + 
                 ",\"gyro\":" + String(pitchGyro, 1) + 
                 ",\"filtered\":" + String(pitchFiltered, 1) + 
                 ",\"alpha\":" + String(alpha, 3) + 
                 ",\"accels\":[";
  for (int i = 0; i < HISTORY_SIZE; i++) {
    int idx = (historyIndex + i) % HISTORY_SIZE;
    json += String(accelHistory[idx], 1);
    if (i < HISTORY_SIZE - 1) json += ",";
  }
  json += "],\"gyros\":[";
  for (int i = 0; i < HISTORY_SIZE; i++) {
    int idx = (historyIndex + i) % HISTORY_SIZE;
    json += String(gyroHistory[idx], 1);
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

// HTTP POST endpoint to update alpha
void handlePostTune() {
  if (server.hasArg("alpha")) {
    alpha = server.arg("alpha").toFloat();
    alpha = constrain(alpha, 0.5f, 0.999f);
    Serial.printf("[Filter Tune] Alpha set to: %.3f\n", alpha);
    server.send(200, "text/plain", "OK");
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>IMU Filter Balancer</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 520px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 10px; margin: 20px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 10px; border: 1px solid #334155; text-align: left; }\n";
  html += "  .metric-label { font-size: 9px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 18px; font-weight: bold; margin-top: 5px; color: #38bdf8; font-family: monospace; }\n";
  html += "  .metric-val.filtered { color: #10b981; }\n";
  html += "  .metric-val.gyro { color: #ef4444; }\n";
  html += "  canvas { background-color: #0f172a; border-radius: 8px; border: 1px solid #334155; margin-top: 15px; width: 100%; aspect-ratio: 2/1; }\n";
  html += "  .tuning-box { border-top: 2px solid #334155; padding-top: 15px; margin-top: 20px; text-align: left; }\n";
  html += "  .slider-row { display: flex; align-items: center; gap: 15px; margin-bottom: 12px; }\n";
  html += "  label { font-weight: bold; color: #94a3b8; width: 160px; font-size: 13px; }\n";
  html += "  input[type=range] { flex-grow: 1; height: 8px; border-radius: 4px; background: #334155; outline: none; -webkit-appearance: none; }\n";
  html += "  input[type=range]::-webkit-slider-thumb { -webkit-appearance: none; width: 18px; height: 18px; border-radius: 50%; background: #38bdf8; cursor: pointer; }\n";
  html += "  .value-lbl { font-family: monospace; font-size: 15px; font-weight: bold; width: 60px; text-align: right; color: #38bdf8; }\n";
  html += "  .footer { text-align: center; margin-top: 25px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>IMU Complementary Filter</h1>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Accel Pitch</div><div class=\"metric-val\" id=\"accDisplay\">0.0&deg;</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Gyro Pitch</div><div class=\"metric-val gyro\" id=\"gyrDisplay\">0.0&deg;</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Filtered Pitch</div><div class=\"metric-val filtered\" id=\"fltDisplay\">0.0&deg;</div></div>\n";
  html += "  </div>\n";
  
  html += "  <canvas id=\"chartCanvas\" width=\"400\" height=\"200\"></canvas>\n";
  
  html += "  <div class=\"tuning-box\">\n";
  html += "    <h3>Filter Calibration</h3>\n";
  html += "    <div class=\"slider-row\"><label>Filter Coefficient (Alpha)</label><input type=\"range\" id=\"alphaSldr\" min=\"0.80\" max=\"0.99\" step=\"0.01\" value=\"" + String(alpha, 2) + "\" oninput=\"tuneFilter()\"><span id=\"alphaVal\" class=\"value-lbl\">" + String(alpha, 2) + "</span></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Blue: Accel (Noisy) | Red: Gyro (Drift) | Green: Filtered</p>\n";
  html += "</div>\n";
  
  // JavaScript multi-line canvas graph
  html += "<script>\n";
  html += "  const canvas = document.getElementById('chartCanvas');\n";
  html += "  const ctx = canvas.getContext('2d');\n";
  
  html += "  function drawChart(accels, gyros, filtereds) {\n";
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
  
  html += "    if (accels.length < 2) return;\n";
  html += "    const step = canvas.width / (accels.length - 1);\n";
  
  // Plot Accel (Blue)
  html += "    ctx.strokeStyle = '#38bdf8';\n";
  html += "    ctx.lineWidth = 1.5;\n";
  html += "    ctx.beginPath();\n";
  html += "    accels.forEach((val, i) => {\n";
  html += "      const x = i * step;\n";
  html += "      const y = 100 - (val * 1.5); // Scaled around center line\n";
  html += "      if (i === 0) ctx.moveTo(x, y);\n";
  html += "      else ctx.lineTo(x, y);\n";
  html += "    });\n";
  html += "    ctx.stroke();\n";
  
  // Plot Gyro (Red)
  html += "    ctx.strokeStyle = '#ef4444';\n";
  html += "    ctx.lineWidth = 1.5;\n";
  html += "    ctx.beginPath();\n";
  html += "    gyros.forEach((val, i) => {\n";
  html += "      const x = i * step;\n";
  html += "      const y = 100 - (val * 1.5);\n";
  html += "      if (i === 0) ctx.moveTo(x, y);\n";
  html += "      else ctx.lineTo(x, y);\n";
  html += "    });\n";
  html += "    ctx.stroke();\n";
  
  // Plot Filtered (Green)
  html += "    ctx.strokeStyle = '#10b981';\n";
  html += "    ctx.lineWidth = 3.0;\n";
  html += "    ctx.beginPath();\n";
  html += "    filtereds.forEach((val, i) => {\n";
  html += "      const x = i * step;\n";
  html += "      const y = 100 - (val * 1.5);\n";
  html += "      if (i === 0) ctx.moveTo(x, y);\n";
  html += "      else ctx.lineTo(x, y);\n";
  html += "    });\n";
  html += "    ctx.stroke();\n";
  html += "  }\n";
  
  html += "  function updateHUD() {\n";
  html += "    fetch('/api/data')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('accDisplay').innerText = data.accel.toFixed(1) + '°';\n";
  html += "        document.getElementById('gyrDisplay').innerText = data.gyro.toFixed(1) + '°';\n";
  html += "        document.getElementById('fltDisplay').innerText = data.filtered.toFixed(1) + '°';\n";
  html += "        drawChart(data.accels, data.gyros, data.filtereds);\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  let lastSentTime = 0;\n";
  html += "  function tuneFilter() {\n";
  html += "    const alphaVal = document.getElementById('alphaSldr').value;\n";
  html += "    document.getElementById('alphaVal').innerText = parseFloat(alphaVal).toFixed(2);\n";
  
  html += "    const now = Date.now();\n";
  html += "    if (now - lastSentTime < 100) return;\n";
  html += "    lastSentTime = now;\n";
  
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('alpha', alphaVal);\n";
  
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
  
  Wire.begin();
  if (!mpu.begin()) {
    Serial.println("[IMU Error] MPU-6050 sensor missing! Running in simulation mode.");
  }
  
  // Initialize history buffers
  for (int i = 0; i < HISTORY_SIZE; i++) {
    accelHistory[i] = 0.0;
    gyroHistory[i] = 0.0;
    filteredHistory[i] = 0.0;
  }
  
  Serial.println("\nESP32 IMU Balancer Node starting...");
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
  
  lastFilterTime = millis();
}

void loop() {
  server.handleClient();
  
  // Periodically compute filter state
  computeComplementaryFilter();
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. In simulation, since the MPU-6050 is simulated using built-in drift functions, click **Run** to launch.
3. Open your web browser and navigate to the printed IP address.
4. Verify that the line chart shows three lines: Blue (noisy accel pitch), Red (drifting gyro pitch), and Green (clean merged pitch).
5. Drag the alpha slider on the webpage. Set it to `0.98`. Verify that the green filtered line becomes extremely smooth and filters out the noise.

## Expected Output
Serial Monitor:
```
[IMU Error] MPU-6050 sensor missing! Running in simulation mode.
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
```

Browser JSON Console (`/api/data`):
```json
{"accel":12.5,"gyro":35.2,"filtered":13.1,"alpha":0.960,"accels":[12,13,12.5],"gyros":[32,34,35.2],"filtereds":[12.8,13.0,13.1]}
```

## Expected Canvas Behavior
* The simulated IMU drift curves generate real-time comparison lines on the webpage graph.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pitchAccel = atan2(...)` | Computes the pitch angle in degrees using trigonometry on gravity vectors. |
| `pitchGyro += gyroY * dt` | Integrates gyroscope angular rates over time to track angle. |
| `alpha * (angle + gyro * dt) + (1-alpha) * accel` | Merges gyro and accel angles to eliminate noise and drift. |

## Hardware & Safety Concept: Sensor Fusion and Balance Control Loops
* **Sensor Fusion**: The Complementary Filter is a simple and effective sensor fusion algorithm. The gyroscope is highly accurate in the short term but suffers from integration drift. The accelerometer is noisy in the short term due to movement vibration but stable in the long term because it references gravity. Combining them using a high weight for the gyro (`alpha = 0.98`) and a low weight for the accel (`0.02`) provides a clean, drift-free angle estimate.
* **Balance Control Loops**: In self-balancing robots (like Segways), a complementary filter provides the feedback angle to a PID motor loop (Project 152). If the filter response is too slow, the robot will fall over before the motors can correct. Keep the filter loop executing at high frequency (at least 200 Hz / 5 ms).

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a balance bar representing the pitch angle.
2. **Tilt alarm buzzer**: Add a buzzer (GPIO 15) to sound an alarm if the filtered angle exceeds 30 degrees.
3. **Roll Axis**: Modify the code to calculate and plot both pitch and roll axes simultaneously.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| MPU-6050 fails to initialize | SDA/SCL lines reversed | Check that SDA is connected to GPIO 21 and SCL to GPIO 22 |
| The gyro line drifts completely off-screen | Integration drift | This is normal for raw gyros. The complementary filter corrects this. Verify the green line is stable |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [105 - ESP32 Thermostat Controller with Web Adjustments](105-esp32-thermostat-controller-iot-ntc-relay-lcd-web-threshold-adjust.md)
- [154 - ESP32 Web Page Kalman Filter Graph](154-esp32-web-page-kalman-filter-graph.md)
- [156 - Web Page Sound Level FFT analyzer](156-esp32-web-page-sound-level-fft-analyzer.md) (Next project)
