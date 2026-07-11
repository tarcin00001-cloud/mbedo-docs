# 166 - Smart Fan with Web Temperature Threshold adjustment

Build an automated cooling fan station on the ESP32 that samples a DHT22 climate sensor on GPIO 14, drives a DC motor fan speed using LEDC hardware PWM on GPIO 12, and hosts a web dashboard with status displays and adjustable temperature threshold sliders.

## Goal
Learn how to map sensor readings dynamically to PWM duty cycles, configure hardware LEDC controllers, build real-time web dashboards, and parse HTTP POST settings.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A DHT22 sensor is on GPIO 14, and a DC fan motor on GPIO 12. The ESP32 monitors temperature. Below a user-defined minimum threshold (e.g. 25 °C), the fan remains OFF. As the temperature rises toward a maximum threshold (e.g. 35 °C), the fan speed increases proportionally from 30% to 100% duty cycle. A webpage displays climate metrics and features sliders to adjust thresholds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temperature & Humidity Sensor | `dht22` | Yes | Yes |
| DC Fan Motor (with transistor/driver)| `motor_dc` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | OUT (Signal) | GPIO14 | Yellow | Climate sensor signal |
| DC Motor Driver| PWM (Gate/IN1)| GPIO12 | Blue | Speed control signal |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a flyback diode (like 1N4007) across the DC motor terminals to absorb voltage spikes during switching.

## Code
```cpp
// Smart Fan with Web Temperature (DHT22 climate sensor + Proportional LEDC PWM + Web Sliders)
#include <WiFi.h>
#include <WebServer.h>
#include <DHT.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int DHT_PIN = 14;
#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

const int MOTOR_PIN = 12;

// LEDC PWM Configuration
const int PWM_CHAN = 0;
const int PWM_FREQ = 5000;
const int PWM_RES = 8; // 8-bit resolution (0-255)

WebServer server(80);

// Global climate and control variables
float ambientTemp = 24.0;
float ambientHumid = 50.0;
int motorSpeedPwm = 0;

// User-adjustable thresholds
float tempMinThreshold = 25.0; // Fan starts at minimum speed
float tempMaxThreshold = 35.0; // Fan runs at 100% speed

unsigned long lastSampleTime = 0;
const unsigned long SAMPLE_INTERVAL_MS = 2000; // Sample DHT22 every 2 seconds

// Calculate fan speed based on climate thresholds
void updateSmartFan() {
  unsigned long now = millis();
  if (now - lastSampleTime >= SAMPLE_INTERVAL_MS) {
    lastSampleTime = now;
    
    // 1. Sample temperature and humidity
    float t = dht.readTemperature();
    float h = dht.readHumidity();
    
    if (!isnan(t) && !isnan(h)) {
      ambientTemp = t;
      ambientHumid = h;
    }
    
    // 2. Proportional Speed Mapping
    if (ambientTemp < tempMinThreshold) {
      motorSpeedPwm = 0; // Fan OFF
    } 
    else if (ambientTemp >= tempMaxThreshold) {
      motorSpeedPwm = 255; // Full speed
    } 
    else {
      // Linearly map speed between 30% (76 PWM) and 100% (255 PWM)
      float factor = (ambientTemp - tempMinThreshold) / (tempMaxThreshold - tempMinThreshold);
      motorSpeedPwm = 76 + (int)(factor * (255 - 76));
    }
    
    // 3. Drive DC Motor
    ledcWrite(PWM_CHAN, motorSpeedPwm);
    
    Serial.printf("[Climate] Temp: %.1f C | Humidity: %.1f%% | Fan Speed: %d%%\n", 
                  ambientTemp, ambientHumid, (motorSpeedPwm * 100) / 255);
  }
}

// HTTP API endpoint returning JSON data
void handleGetStatus() {
  String json = "{\"temp\":" + String(ambientTemp, 1) + 
                 ",\"humidity\":" + String(ambientHumid, 1) + 
                 ",\"speed\":" + String((motorSpeedPwm * 100) / 255) + 
                 ",\"min\":" + String(tempMinThreshold, 1) + 
                 ",\"max\":" + String(tempMaxThreshold, 1) + "}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to update parameters
void handlePostThresholds() {
  if (server.hasArg("min") && server.hasArg("max")) {
    float newMin = server.arg("min").toFloat();
    float newMax = server.arg("max").toFloat();
    
    // Safety check: ensure min is below max threshold
    if (newMin < newMax) {
      tempMinThreshold = newMin;
      tempMaxThreshold = newMax;
      Serial.printf("[Threshold Config] Min set to: %.1f, Max set to: %.1f\n", tempMinThreshold, tempMaxThreshold);
      server.send(200, "text/plain", "OK");
    } else {
      server.send(400, "text/plain", "Error: Min must be less than Max");
    }
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Smart Fan Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 480px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .speed-val { font-size: 56px; font-weight: 800; font-family: monospace; color: #10b981; margin: 15px 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 25px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 12px; border: 1px solid #334155; text-align: left; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 22px; font-weight: bold; margin-top: 5px; color: #38bdf8; font-family: monospace; }\n";
  html += "  .tuning-box { border-top: 2px solid #334155; padding-top: 20px; margin-top: 25px; text-align: left; }\n";
  html += "  .slider-row { display: flex; align-items: center; gap: 15px; margin-bottom: 15px; }\n";
  html += "  label { font-weight: bold; color: #94a3b8; width: 150px; font-size: 13px; }\n";
  html += "  input[type=range] { flex-grow: 1; height: 8px; border-radius: 4px; background: #334155; outline: none; -webkit-appearance: none; }\n";
  html += "  input[type=range]::-webkit-slider-thumb { -webkit-appearance: none; width: 18px; height: 18px; border-radius: 50%; background: #38bdf8; cursor: pointer; }\n";
  html += "  .value-lbl { font-family: monospace; font-size: 15px; font-weight: bold; width: 55px; text-align: right; color: #38bdf8; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Smart Fan Control</h1>\n";
  html += "  <div id=\"spdDisplay\" class=\"speed-val\">0%</div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Temperature</div><div class=\"metric-val\" id=\"tempDisplay\">0.0 &deg;C</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Humidity</div><div class=\"metric-val\" id=\"humidDisplay\">0.0%</div></div>\n";
  html += "  </div>\n";
  
  html += "  <div class=\"tuning-box\">\n";
  html += "    <h3>Speed Thresholds</h3>\n";
  // Sliders
  html += "    <div class=\"slider-row\"><label>Start Fan Temp</label><input type=\"range\" id=\"minSldr\" min=\"15\" max=\"30\" step=\"0.5\" value=\"" + String(tempMinThreshold, 1) + "\" oninput=\"tuneThresholds()\"><span id=\"minVal\" class=\"value-lbl\">" + String(tempMinThreshold, 1) + "&deg;C</span></div>\n";
  html += "    <div class=\"slider-row\"><label>Full Fan Temp</label><input type=\"range\" id=\"maxSldr\" min=\"31\" max=\"45\" step=\"0.5\" value=\"" + String(tempMaxThreshold, 1) + "\" oninput=\"tuneThresholds()\"><span id=\"maxVal\" class=\"value-lbl\">" + String(tempMaxThreshold, 1) + "&deg;C</span></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">PWM Frequency: 5 kHz | 8-bit LEDC resolution</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateHUD() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('spdDisplay').innerText = data.speed + '%';\n";
  html += "        document.getElementById('tempDisplay').innerHTML = data.temp.toFixed(1) + ' &deg;C';\n";
  html += "        document.getElementById('humidDisplay').innerText = data.humidity.toFixed(1) + '%';\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  let lastSentTime = 0;\n";
  html += "  function tuneThresholds() {\n";
  html += "    const min = document.getElementById('minSldr').value;\n";
  html += "    const max = document.getElementById('maxSldr').value;\n";
  
  html += "    document.getElementById('minVal').innerHTML = min + '&deg;C';\n";
  html += "    document.getElementById('maxVal').innerHTML = max + '&deg;C';\n";
  
  html += "    const now = Date.now();\n";
  html += "    if (now - lastSentTime < 100) return;\n";
  html += "    lastSentTime = now;\n";
  
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('min', min);\n";
  html += "    body.append('max', max);\n";
  
  html += "    fetch('/api/threshold', { method: 'POST', body: body });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateHUD();\n";
  html += "    setInterval(updateHUD, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  dht.begin();
  
  // Configure LEDC PWM Pin
  ledcSetup(PWM_CHAN, PWM_FREQ, PWM_RES);
  ledcAttachPin(MOTOR_PIN, PWM_CHAN);
  ledcWrite(PWM_CHAN, 0); // Start OFF
  
  Serial.println("\nESP32 Smart Fan starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/status", handleGetStatus);
  server.on("/api/threshold", HTTP_POST, handlePostThresholds);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // Periodically refresh sensor calculations and speed controls
  updateSmartFan();
  
  delay(5);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22 Climate Sensor**, and **DC Motor** onto the canvas.
2. Wire DHT22 output to **GPIO14** and Motor speed pin to **GPIO12**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Set simulated DHT22 temperature to 24.0 °C. Verify that the fan does not spin (0% speed).
6. Slide simulated temperature to 30.0 °C. Verify that the DC motor widget spins at a moderate speed and the webpage indicates speed percentage.
7. Slide simulated temperature to 36.0 °C. Verify that the fan spins at maximum speed.
8. Adjust the start temperature threshold on the webpage to 31.0 °C. Verify that the fan turns OFF since temperature is now below the threshold.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Climate] Temp: 30.0 C | Humidity: 55.2% | Fan Speed: 64%
```

## Expected Canvas Behavior
* Adjusting the simulated temperature slider changes the rotational velocity of the DC motor widget dynamically.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `dht.readTemperature()` | Samples ambient temperature from the climate sensor. |
| `factor = (ambientTemp - tempMinThreshold) / ...` | Computes proportional scaling ratio. |
| `ledcWrite(PWM_CHAN, motorSpeedPwm)` | Drives the DC motor using the mapped PWM duty cycle. |

## Hardware & Safety Concept: Kickback Diodes and Transistor Drivers
* **Kickback Diodes (Flyback Diodes)**: Inductive loads (like DC motor coils) store magnetic energy when active. When the driving transistor turns off, this energy collapses, generating a large negative voltage spike (often exceeding 100V). Solder a flyback diode (such as a 1N4007) in parallel with the motor terminals (cathode to positive side, anode to negative side) to clamp these spikes.
* **Transistor Drivers**: The ESP32 GPIO pins can only source about 20–40 mA of current. A DC motor typically requires 100–500 mA. Connecting a motor directly to an ESP32 pin will burn out the microcontroller. Always use a MOSFET (like an IRF520) or an NPN transistor (like a PN2222) to switch the high current required by the motor.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a progress bar indicating fan speed.
2. **Humid Threshold Trigger**: Modify the code to turn the fan ON at 100% speed if humidity exceeds 85% (anti-condensation switch).
3. **Speed decay delay**: Add logic to keep the fan running at minimum speed for at least 30 seconds after temperature drops below the threshold to prevent short-cycling.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Fan motor hums but does not spin | PWM frequency too high or duty cycle too low | DC motors require a minimum starting voltage. Ensure the minimum mapped speed is at least 30% (76 PWM) and decrease the PWM frequency |
| Temperature readings show NaN | Pull-up resistor missing | Ensure the DHT22 signal line has a 10 kΩ pull-up resistor connected to the 3.3V line |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [67 - ESP32 Web Page DHT22 climate display](67-esp32-web-page-dht22-climate-display.md)
- [100 - Smart Fan (MbedO Curriculum)](100-smart-fan.md)
- [165 - Smart Door Lock with Web Access Override logs](165-esp32-smart-door-lock-with-web-access-override-logs.md)
- [167 - Current Overload Contactor Breaker IoT Dashboard](167-esp32-current-overload-contactor-breaker-iot-dashboard-reset-online.md) (Next project)
