# 160 - Ultrasonic Wind Anemometer IoT Dashboard

Build an advanced ultrasonic wind anemometer station on the ESP32 that measures wind vectors using two HC-SR04 ultrasonic sensors (Trig1/Echo1 on GPIO 12/13 representing North-South vector, Trig2/Echo2 on GPIO 14/15 representing East-West vector), computes total wind speed and direction, and hosts a web dashboard with a dynamic circular wind compass gauge.

## Goal
Learn how to capture multiple ultrasonic sensors, perform trigonometric vector additions, compute wind angles, render circular gauge dials on HTML5 Canvas, and serve web interfaces.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Two HC-SR04 sensors are connected on GPIOs. The ESP32 triggers pulses. By measuring differences in transit times (simulated by sensor distance shifts), it calculates wind vectors along N-S and E-W axes. It computes total wind speed (`m/s`) and direction (`degrees`). Navigating to the ESP32's IP address displays a compass gauge showing wind velocity and vector angles.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HC-SR04 Ultrasonic Sensors | `ultrasonic` | Yes (2 pieces) | Yes (2 pieces) |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use the sliders on the two ultrasonic widgets to adjust the simulated transit times, representing changes in wind velocity along N-S and E-W axes.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 (N-S) | TRIG / ECHO | GPIO12 / GPIO13 | Yellow / Green | North-South vector sensor |
| HC-SR04 (E-W) | TRIG / ECHO | GPIO14 / GPIO15 | Orange / Blue | East-West vector sensor |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power both HC-SR04 sensors from the 5V Vin rail.

## Code
```cpp
// Ultrasonic Wind Anemometer IoT Dashboard (Dual Vector calculations + HTML5 Compass Gauge)
#include <WiFi.h>
#include <WebServer.h>
#include <math.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Sensor 1: North-South Vector
const int TRIG1 = 12;
const int ECHO1 = 13;

// Sensor 2: East-West Vector
const int TRIG2 = 14;
const int ECHO2 = 15;

WebServer server(80);

float windSpeed = 0.0;     // Total speed (m/s)
float windDirection = 0.0; // Compass direction (0 - 360 degrees)
float windNS = 0.0;        // N-S component
float windEW = 0.0;        // E-W component

unsigned long lastSampleTime = 0;
const unsigned long SAMPLE_INTERVAL_MS = 250; // Sample 4 times per second

// Read distance from HC-SR04 in cm
float readDistanceCm(int trigPin, int echoPin) {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  
  long duration = pulseIn(echoPin, HIGH, 20000);
  if (duration == 0) return 20.0; // Baseline
  
  return (float)duration * 0.0343 / 2.0;
}

// Calculate wind vectors from sensor differences
void calculateWindVectors() {
  // Read distance offsets (representing sound travel times)
  float distNS = readDistanceCm(TRIG1, ECHO1);
  float distEW = readDistanceCm(TRIG2, ECHO2);
  
  // Convert distance offsets from center (20cm) to wind speed components (m/s)
  // Shift center: e.g. 20cm represents 0 m/s wind.
  windNS = (distNS - 20.0) * 3.0; // Mapped scaling factor
  windEW = (distEW - 20.0) * 3.0;
  
  // 1. Calculate Total Wind Speed = sqrt(NS^2 + EW^2)
  windSpeed = sqrt(windNS * windNS + windEW * windEW);
  
  // Filter trace noise
  if (windSpeed < 0.2) windSpeed = 0.0;
  
  // 2. Calculate Wind Direction Angle (Degrees) = atan2(EW, NS)
  float angleRad = atan2(windEW, windNS);
  windDirection = angleRad * 180.0 / PI;
  if (windDirection < 0) {
    windDirection += 360.0; // Normalize angle to 0 - 360 degrees
  }
}

// HTTP API endpoint returning JSON data
void handleGetWind() {
  String json = "{\"speed\":" + String(windSpeed, 2) + 
                 ",\"direction\":" + String(windDirection, 1) + 
                 ",\"ns\":" + String(windNS, 2) + 
                 ",\"ew\":" + String(windEW, 2) + "}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Anemometer Dashboard</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 480px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .speed-display { font-size: 48px; font-weight: 800; font-family: monospace; color: #10b981; margin: 15px 0; }\n";
  html += "  canvas { background-color: #0f172a; border-radius: 50%; border: 2px solid #334155; margin: 20px auto; display: block; width: 220px; height: 220px; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin-top: 15px; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 12px; border: 1px solid #334155; text-align: left; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 20px; font-weight: bold; margin-top: 5px; color: #38bdf8; font-family: monospace; }\n";
  html += "  .footer { text-align: center; margin-top: 25px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Ultrasonic Anemometer</h1>\n";
  html += "  <div id=\"speedDisplay\" class=\"speed-display\">0.0 m/s</div>\n";
  
  // Compass dial canvas
  html += "  <canvas id=\"compassCanvas\" width=\"200\" height=\"200\"></canvas>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Wind Direction</div><div class=\"metric-val\" id=\"dirDisplay\">0.0&deg;</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">N-S / E-W Vector</div><div class=\"metric-val\" id=\"vecDisplay\">0.0 / 0.0</div></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Transit time vector addition | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript canvas compass rendering script
  html += "<script>\n";
  html += "  const canvas = document.getElementById('compassCanvas');\n";
  html += "  const ctx = canvas.getContext('2d');\n";
  
  html += "  function drawCompass(direction, speed) {\n";
  html += "    ctx.clearRect(0, 0, canvas.width, canvas.height);\n";
  html += "    const cx = canvas.width / 2;\n";
  html += "    const cy = canvas.height / 2;\n";
  html += "    const r = cx - 15;\n";
  
  // Draw outer ring ticks
  html += "    ctx.strokeStyle = '#334155';\n";
  html += "    ctx.lineWidth = 2;\n";
  html += "    ctx.beginPath();\n";
  html += "    ctx.arc(cx, cy, r, 0, 2*Math.PI);\n";
  html += "    ctx.stroke();\n";
  
  // Cardinal Letters (N, S, E, W)
  html += "    ctx.fillStyle = '#64748b';\n";
  html += "    ctx.font = 'bold 12px sans-serif';\n";
  html += "    ctx.textAlign = 'center';\n";
  html += "    ctx.textBaseline = 'middle';\n";
  html += "    ctx.fillText('N', cx, cy - r + 10);\n";
  html += "    ctx.fillText('S', cx, cy + r - 10);\n";
  html += "    ctx.fillText('E', cx + r - 10, cy);\n";
  html += "    ctx.fillText('W', cx - r + 10, cy);\n";
  
  // Draw wind vector arrow pointing in direction of wind travel
  if (speed > 0) {\n";
  html += "      const angleRad = (direction * Math.PI) / 180.0;\n";
  // Invert arrow direction to show where wind is blowing FROM
  html += "      const arrowX = cx - (r - 25) * Math.sin(angleRad);\n";
  html += "      const arrowY = cy - (r - 25) * Math.cos(angleRad);\n";
  
  html += "      ctx.strokeStyle = '#ef4444';\n";
  html += "      ctx.lineWidth = 4;\n";
  html += "      ctx.beginPath();\n";
  html += "      ctx.moveTo(cx, cy);\n";
  html += "      ctx.lineTo(arrowX, arrowY);\n";
  html += "      ctx.stroke();\n";
  
  // Arrow head
  html += "      ctx.fillStyle = '#ef4444';\n";
  html += "      ctx.beginPath();\n";
  html += "      ctx.arc(arrowX, arrowY, 6, 0, 2*Math.PI);\n";
  html += "      ctx.fill();\n";
  html += "    }\n";
  
  // Center hub dot
  html += "    ctx.fillStyle = '#38bdf8';\n";
  html += "    ctx.beginPath();\n";
  html += "    ctx.arc(cx, cy, 5, 0, 2*Math.PI);\n";
  html += "    ctx.fill();\n";
  html += "  }\n";
  
  html += "  function updateHUD() {\n";
  html += "    fetch('/api/wind')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('speedDisplay').innerText = data.speed.toFixed(1) + ' m/s';\n";
  html += "        document.getElementById('dirDisplay').innerHTML = data.direction.toFixed(1) + '&deg;';\n";
  html += "        document.getElementById('vecDisplay').innerText = data.ns.toFixed(1) + ' / ' + data.ew.toFixed(1);\n";
  html += "        drawCompass(data.direction, data.speed);\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateHUD();\n";
  html += "    setInterval(updateHUD, 300); // Polling index\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(TRIG1, OUTPUT);
  pinMode(ECHO1, INPUT);
  pinMode(TRIG2, OUTPUT);
  pinMode(ECHO2, INPUT);
  
  Serial.println("\nESP32 Wind Anemometer starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/wind", handleGetWind);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  lastSampleTime = millis();
}

void loop() {
  server.handleClient();
  
  // Sample vectors periodically (every 250 ms)
  unsigned long now = millis();
  if (now - lastSampleTime >= SAMPLE_INTERVAL_MS) {
    lastSampleTime = now;
    calculateWindVectors();
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **2 HC-SR04 Sensors** onto the canvas.
2. Wire Sensor 1 (N-S) TRIG/ECHO to **GPIO12/GPIO13**.
3. Wire Sensor 2 (E-W) TRIG/ECHO to **GPIO14/GPIO15**.
4. Paste the code and click **Run**.
5. Open your web browser and navigate to the printed IP address.
6. Set simulated Sensor 1 (N-S) to 25 cm, and Sensor 2 (E-W) to 15 cm.
7. Verify that the webpage compass arrow points in the calculated direction, and the speed indicates wind velocity.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
```

Browser JSON Console (`/api/wind`):
```json
{"speed":18.03,"direction":303.7,"ns":15.00,"ew":-10.00}
```

## Expected Canvas Behavior
* Adjusting the distance sliders on the simulated ultrasonic sensor widgets updates the wind vector direction and arrow on the webpage compass.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `readDistanceCm(...)` | Measures transit time along a specific coordinate vector axis. |
| `windNS` / `windEW` | Mapped scaling variables representing wind speeds along axial coordinates. |
| `sqrt(...)` | Calculates the total resultant speed vector using the Pythagorean theorem. |
| `atan2(...)` | Computes the resultant wind direction angle in radians. |

## Hardware & Safety Concept: Sound Speed Temp Dependencies and Transducer Layouts
* **Speed of Sound Temperature Dependency**: The speed of sound in air varies with temperature: `c ≈ 331.3 + 0.606 * TempC` (m/s). If not compensated, temperature shifts will introduce measurement errors in the wind speed calculation. In production, read a temperature sensor (like DHT22) to adjust the transit time reference.
* **Transducers**: Mechanical cups can freeze or jam. Ultrasonic anemometers have no moving parts, making them highly reliable. Align the transducers carefully in a rigid cross structure to ensure the acoustic paths are perpendicular.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a small compass arrow locally.
2. **High Wind Relay Trigger**: Add a relay (GPIO 2) to trigger an emergency cutoff switch if the wind speed exceeds 15 m/s.
3. **SPIFFS integration**: Log wind speed averages to a file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Speed reads zero constantly | Sensor pins reversed | Verify TRIG and ECHO connections for both sensors |
| Wind direction is inverted | Vector math wrong | Check that the `atan2` parameters are structured as `atan2(windEW, windNS)` |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [124 - ESP32 Obstacle Avoidance Robot with Web HUD](124-esp32-obstacle-avoidance-robot-with-web-hud-telemetry.md)
- [159 - ESP32 Automatic Battery Capacity Tester](159-esp32-automatic-battery-capacity-tester-iot-web-logger.md)
- [161 - Solar Tracker Dual Axis Controller IoT Status Dashboard](161-esp32-solar-tracker-dual-axis-controller-iot-status-dashboard.md) (Next project)
