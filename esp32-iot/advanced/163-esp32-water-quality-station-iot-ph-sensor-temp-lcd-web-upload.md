# 163 - Water Quality Station IoT (pH sensor + Temp + LCD + Web upload)

Build an environmental water quality telemetry station on the ESP32 that reads an analog pH sensor on GPIO 34 and a DS18B20 temperature sensor on GPIO 12, displays live data locally on an I2C LCD 1602 (GPIO 21/22), and uploads measurements to a cloud database using secure HTTP REST POST requests.

## Goal
Learn how to read pH sensors, interface DS18B20 temperature probes over OneWire, write to I2C LCDs, compile JSON payloads, and post data to cloud REST APIs.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A pH probe is on GPIO 34, a DS18B20 sensor on GPIO 12, and an LCD 1602 on I2C. The ESP32 reads water temperature and pH level. It displays the values on the LCD locally. Every 15 seconds, it sends an HTTP POST request containing these parameters to a cloud database simulator (`http://httpbin.org/post`). A local webpage displays live values and sync logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| pH Sensor Module & Probe | `potentiometer` | Yes | Yes |
| DS18B20 Waterproof Temperature Sensor | `ds18b20` | Yes | Yes |
| I2C LCD 1602 Display | `lcd_i2c` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use a potentiometer to simulate the pH probe output voltage and the simulated DS18B20 temperature slider.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| pH Sensor Board | A0 (Analog Out) | GPIO34 | Yellow | pH signal voltage |
| DS18B20 Sensor | DQ (Data Out) | GPIO12 | Green | OneWire data input |
| LCD 1602 Display| SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C communication |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 4.7 kΩ pull-up resistor between the DS18B20 data (DQ) line and the 3.3V power line.

## Code
```cpp
// Water Quality Station IoT (pH sensor + OneWire Temp + I2C LCD + Web REST upload)
#include <WiFi.h>
#include <WebServer.h>
#include <HTTPClient.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <OneWire.h>
#include <DallasTemperature.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Cloud DB API endpoint
const char* cloudApiUrl = "http://httpbin.org/post";

const int PH_PIN = 34;
const int ONE_WIRE_BUS = 12;

OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

LiquidCrystal_I2C lcd(0x27, 16, 2);
WebServer server(80);

// Global quality metrics
float phValue = 7.0;
float waterTempC = 25.0;

unsigned long lastUploadTime = 0;
const unsigned long UPLOAD_INTERVAL_MS = 15000; // Log and sync every 15 seconds
String cloudSyncStatus = "Idle";

// Read sensors and update local LCD
void updateWaterMetrics() {
  // 1. Read pH Value
  int rawPH = analogRead(PH_PIN);
  float phVoltage = ((float)rawPH / 4095.0) * 3.3;
  
  // Standard pH calibration calculation:
  // Offset represents voltage at neutral pH (7.0), e.g. 1.65V
  // Mapped scale: 1V difference = approx 3.0 pH units
  phValue = 7.0 + (1.65 - phVoltage) * 3.0;
  phValue = constrain(phValue, 0.0, 14.0); // Keep within standard bounds
  
  // 2. Read DS18B20 Temperature
  sensors.requestTemperatures();
  float tempVal = sensors.getTempCByIndex(0);
  if (tempVal > -127.0) { // Check for valid reading
    waterTempC = tempVal;
  } else {
    // Simulation fallback if probe is disconnected
    static float simTemp = 24.5;
    simTemp += ((float)random(-2, 3) / 10.0);
    waterTempC = simTemp;
  }
  
  // 3. Update I2C LCD Local Display
  lcd.setCursor(0, 0);
  lcd.print("pH Value: ");
  lcd.print(phValue, 2);
  lcd.print("   "); // Clear remaining characters
  
  lcd.setCursor(0, 1);
  lcd.print("Water T : ");
  lcd.print(waterTempC, 1);
  lcd.print(" C   ");
}

// Upload telemetry to cloud DB API via HTTP POST
void uploadTelemetryToCloud() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(cloudApiUrl);
    http.addHeader("Content-Type", "application/json");
    
    // Compile JSON payload
    String jsonPayload = "{\"ph\":" + String(phValue, 2) + 
                         ",\"temperature\":" + String(waterTempC, 1) + 
                         ",\"status\":\"" + (phValue < 6.0 || phValue > 8.0 ? "WARNING" : "NORMAL") + "\"}";
                         
    Serial.printf("[Cloud] Pushing payload -> %s\n", jsonPayload.c_str());
    int httpResponseCode = http.POST(jsonPayload);
    
    if (httpResponseCode == 200 || httpResponseCode == 201) {
      cloudSyncStatus = "Success (Code: " + String(httpResponseCode) + ")";
      Serial.println("[Cloud] Upload successful.");
    } else {
      cloudSyncStatus = "Failed (Code: " + String(httpResponseCode) + ")";
      Serial.printf("[Cloud Error] POST failed: %d\n", httpResponseCode);
    }
    http.end();
  } else {
    cloudSyncStatus = "WiFi Offline";
  }
}

// HTTP API endpoint returning JSON data
void handleGetStatus() {
  String json = "{\"ph\":" + String(phValue, 2) + 
                 ",\"temp\":" + String(waterTempC, 1) + 
                 ",\"sync\":\"" + cloudSyncStatus + "\"}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Water Quality Dashboard</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 480px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 25px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; text-align: left; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 24px; font-weight: bold; margin-top: 5px; color: #10b981; font-family: monospace; }\n";
  html += "  .metric-val.warning { color: #f59e0b; }\n";
  html += "  .badge { display: inline-block; padding: 6px 12px; border-radius: 12px; font-size: 11px; font-weight: bold; text-transform: uppercase; margin-bottom: 15px; background-color: #334155; color: #94a3b8; }\n";
  html += "  .badge.synced { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Water Quality Station</h1>\n";
  html += "  <div id=\"syncBadge\" class=\"badge\">Idle</div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">pH Level</div><div class=\"metric-val\" id=\"phDisplay\">7.00</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Temperature</div><div class=\"metric-val\" id=\"tempDisplay\">0.0 &deg;C</div></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">I2C LCD: 0x27 | Cloud Target: httpbin.org | Upload: 15s</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateQualityHUD() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        const phEl = document.getElementById('phDisplay');\n";
  html += "        phEl.innerText = data.ph.toFixed(2);\n";
  
  // Visual coloring: green if neutral (6.5 to 7.5), orange if acidic/alkaline
  html += "        if (data.ph < 6.5 || data.ph > 7.5) {\n";
  html += "          phEl.className = 'metric-val warning';\n";
  html += "        } else {\n";
  html += "          phEl.className = 'metric-val';\n";
  html += "        }\n";
  
  html += "        document.getElementById('tempDisplay').innerHTML = data.temp.toFixed(1) + ' &deg;C';\n";
  
  html += "        const bdgEl = document.getElementById('syncBadge');\n";
  html += "        bdgEl.innerText = data.sync;\n";
  html += "        if (data.sync.startsWith('Success')) {\n";
  html += "          bdgEl.className = 'badge synced';\n";
  html += "        } else {\n";
  html += "          bdgEl.className = 'badge';\n";
  html += "        }\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateQualityHUD();\n";
  html += "    setInterval(updateQualityHUD, 2000);\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Initialize OneWire probes
  sensors.begin();
  
  // Initialize I2C LCD
  lcd.init();
  lcd.backlight();
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Water Quality");
  lcd.setCursor(0, 1);
  lcd.print("Booting system...");
  
  pinMode(PH_PIN, INPUT);
  
  Serial.println("\nESP32 Water Quality Station starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/status", handleGetStatus);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // Periodically refresh data and upload to cloud (every 15 seconds)
  updateWaterMetrics();
  
  unsigned long now = millis();
  if (now - lastUploadTime >= UPLOAD_INTERVAL_MS) {
    lastUploadTime = now;
    uploadTelemetryToCloud();
  }
  
  delay(50); // Small interval delay
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DS18B20 Temp Probe**, **I2C LCD 1602**, and a **Potentiometer** (to simulate pH) onto the canvas.
2. Wire Potentiometer to **GPIO34**, DS18B20 DQ to **GPIO12**, and LCD to **GPIO21/22**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Slide the simulated pH potentiometer to 50% (neutral 1.65V). Verify that the LCD reads `pH Value: 7.00` and the webpage displays green.
6. Slide the potentiometer to 90% (acidic). Verify that the LCD and webpage update to reflect the change, and the status changes to yellow.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Cloud] Pushing payload -> {"ph":7.0,"temperature":25.0,"status":"NORMAL"}
[Cloud] Upload successful.
```

## Expected Canvas Behavior
* Adjusting the simulated pH potentiometer and temperature slider updates the I2C LCD character display and the webpage diagnostics.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `sensors.requestTemperatures()` | Commands all DS18B20 sensors on the OneWire bus to perform conversion. |
| `phValue = 7.0 + (1.65 - phVoltage) * 3.0` | Maps analog sensor input voltages to equivalent pH values. |
| `lcd.print(phValue, 2)` | Outputs the calculated pH value to the I2C LCD. |
| `http.POST(jsonPayload)` | Initiates the HTTP POST request to upload data to the cloud service. |

## Hardware & Safety Concept: pH Probe Impedance and Calibration
* **pH Probe Impedance**: pH probes generate very small voltages with high internal resistance (impedance). Connecting them directly to the ESP32 ADC will pull down the signal, returning incorrect values. Always connect the probe to a signal conditioning circuit (like a pH meter module containing a CA3140 op-amp) to buffer the high-impedance voltage.
* **Calibration**: Probes drift based on temperature. To calibrate, place the probe in a neutral buffer solution (pH 7.0) and adjust the potentiometer on the amplifier board until the output voltage reads exactly 1.65V. Repeat the step in an acidic buffer (pH 4.01) to calibrate the slope factor in your code.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a real-time graph of the pH level.
2. **Dosing Pump Control**: Add a relay (GPIO 2) to control an alkaline dosing pump, activating it if the pH drops below 6.0 (acidic) to neutralize the water.
3. **SPIFFS integration**: Log water parameters to a file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD display stays blank | Contrast potentiometer | Adjust the contrast potentiometer on the back of the I2C backpack board |
| Temperature reads -127 °C | Pull-up resistor missing | Ensure a 4.7 kΩ pull-up resistor is connected between the DQ line (GPIO 12) and the 3.3V power rail |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [67 - ESP32 Web Page DHT22 climate display](67-esp32-web-page-dht22-climate-display.md)
- [139 - ESP32 IoT Weather Station Node](139-esp32-iot-weather-station-node-dht22-bmp180-cloud-database-upload.md)
- [164 - Water Flow Rate Station IoT](164-esp32-water-flow-rate-station-iot-flow-sensor-lcd-web-upload.md) (Next project)
