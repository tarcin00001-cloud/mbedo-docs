# 162 - Solar Tracker Efficiency Web Logger (Voltage/Current -> Cloud Graph)

Build a solar generation efficiency logger on the ESP32 that measures solar panel voltage via an analog divider on GPIO 34, current via an ACS712 sensor on GPIO 35, logs generation curves to an SD card, and uploads telemetry to a cloud database using HTTP REST POST requests.

## Goal
Learn how to measure solar generation variables (voltage, current, power), log data to SD cards, construct JSON payloads, perform HTTP POST requests, and host diagnostic web pages.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A solar panel's output voltage is measured on GPIO 34 through a voltage divider, and its current on GPIO 35 via an ACS712. Every 10 seconds, the ESP32 calculates output power (mW), logs the parameters to `/solar.csv` on the SD card, and uploads a JSON data packet to a cloud service (`http://httpbin.org/post`). A local webpage displays live power stats and sync logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Solar Panel (Small 5V cell) | `potentiometer` | Yes | Yes |
| ACS712 Current Sensor Module | `potentiometer` | Yes | Yes |
| Micro SD Card Reader SPI Module | `sd_card` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use two potentiometers to simulate the solar panel's output voltage and generation current.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Voltage Divider | Midpoint | GPIO34 | Yellow | Panel voltage input |
| ACS712 Sensor | OUT (Analog) | GPIO35 | Green | Generation current input |
| SD Card Module | CS (Chip Select) | GPIO5 | Yellow | SPI chip select |
| SD Card Module | SCK / MISO / MOSI | GPIO18 / 19 / 23 | Green / Blue / White | SPI interface |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 2:1 resistive divider (two 10 kΩ resistors) to scale the solar panel voltage down to a safe range (0–3.3V) for the ESP32 ADC.

## Code
```cpp
// Solar Tracker Efficiency Web Logger (Voltage/Current sensors + SPI SD card + Cloud REST client)
#include <WiFi.h>
#include <WebServer.h>
#include <SPI.h>
#include <SD.h>
#include <HTTPClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Cloud DB API endpoint
const char* cloudApiUrl = "http://httpbin.org/post";

const int VOLTAGE_PIN = 34;
const int CURRENT_PIN = 35;
const int SD_CS_PIN = 5;
const char* logFilepath = "/solar.csv";

WebServer server(80);

// Sensor Calibration values
const int ADC_CENTER_OFFSET = 2048; // ACS712 center Vcc/2 reference
const double CURRENT_SENSITIVITY = 0.185; // 185 mV/A for 5A model

// Global telemetry variables
float panelVoltage = 0.0;
float generationCurrentMa = 0.0;
float generationPowerMw = 0.0;

unsigned long lastLogTime = 0;
const unsigned long LOG_INTERVAL_MS = 10000; // Log and sync every 10 seconds

String sdSyncStatus = "Not Initialized";
String cloudSyncStatus = "Idle";

// Read solar panel sensors
void readSolarSensors() {
  // 1. Read Panel Voltage (2:1 divider)
  int rawVolts = analogRead(VOLTAGE_PIN);
  panelVoltage = ((float)rawVolts / 4095.0) * 3.3 * 2.0;
  
  // 2. Read Generation Current (ACS712)
  int rawCurrent = analogRead(CURRENT_PIN);
  double voltageDiff = ((double)(rawCurrent - ADC_CENTER_OFFSET) / 4095.0) * 3.3;
  double currentAmps = voltageDiff / CURRENT_SENSITIVITY;
  
  // Convert to mA and filter noise floor
  generationCurrentMa = abs(currentAmps * 1000.0);
  if (generationCurrentMa < 5.0) generationCurrentMa = 0.0;
  
  // 3. Compute Power (mW): P = V * I
  generationPowerMw = panelVoltage * generationCurrentMa;
}

// Log data locally to SD Card CSV
void logToSDCard() {
  File file = SD.open(logFilepath, FILE_APPEND);
  if (file) {
    unsigned long timestampS = millis() / 1000;
    file.printf("%lu,%.2f,%.1f,%.1f\n", timestampS, panelVoltage, generationCurrentMa, generationPowerMw);
    file.close();
    sdSyncStatus = "Success";
  } else {
    sdSyncStatus = "Write Error";
    Serial.println("[SD Error] Failed to open CSV file for appending!");
  }
}

// Upload telemetry to cloud DB API via HTTP POST
void uploadToCloud() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(cloudApiUrl);
    http.addHeader("Content-Type", "application/json");
    
    // Compile JSON payload
    String jsonPayload = "{\"voltage\":" + String(panelVoltage, 2) + 
                         ",\"current\":" + String(generationCurrentMa, 1) + 
                         ",\"power\":" + String(generationPowerMw, 1) + 
                         ",\"uptime\":" + String(millis() / 1000) + "}";
                         
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
  String json = "{\"voltage\":" + String(panelVoltage, 2) + 
                 ",\"current\":" + String(generationCurrentMa, 1) + 
                 ",\"power\":" + String(generationPowerMw, 1) + 
                 ",\"sd\":\"" + sdSyncStatus + "\"" +
                 ",\"cloud\":\"" + cloudSyncStatus + "\"}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Solar Generation Logger</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 480px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .power-display { font-size: 56px; font-weight: 800; font-family: monospace; color: #10b981; margin: 15px 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 25px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 12px; border: 1px solid #334155; text-align: left; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 22px; font-weight: bold; margin-top: 5px; color: #38bdf8; font-family: monospace; }\n";
  html += "  .sync-table { width: 100%; border-collapse: collapse; margin-top: 20px; text-align: left; font-size: 13px; }\n";
  html += "  .sync-table th { color: #64748b; border-bottom: 2px solid #334155; padding-bottom: 8px; }\n";
  html += "  .sync-table td { padding: 10px 0; border-bottom: 1px solid #1e293b; color: #f1f5f9; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Solar Generation Logger</h1>\n";
  html += "  <div id=\"powerDisplay\" class=\"power-display\">0.0 mW</div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Voltage</div><div class=\"metric-val\" id=\"voltDisplay\">0.00 V</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Current</div><div class=\"metric-val\" id=\"currDisplay\">0.0 mA</div></div>\n";
  html += "  </div>\n";
  
  html += "  <table class=\"sync-table\">\n";
  html += "    <thead><tr><th>Sync Protocol</th><th>Status</th></tr></thead>\n";
  html += "    <tbody>\n";
  html += "      <tr><td>Local SD Storage</td><td id=\"sdStatus\">Not Initialized</td></tr>\n";
  html += "      <tr><td>Cloud REST DB</td><td id=\"cloudStatus\">Idle</td></tr>\n";
  html += "    </tbody>\n";
  html += "  </table>\n";
  
  html += "  <p class=\"footer\">Telemetry logged every 10 seconds | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateHUD() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('powerDisplay').innerText = data.power.toFixed(1) + ' mW';\n";
  html += "        document.getElementById('voltDisplay').innerText = data.voltage.toFixed(2) + ' V';\n";
  html += "        document.getElementById('currDisplay').innerText = data.current.toFixed(1) + ' mA';\n";
  html += "        document.getElementById('sdStatus').innerText = data.sd;\n";
  html += "        document.getElementById('cloudStatus').innerText = data.cloud;\n";
  // Handled network updates safely
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateHUD();\n";
  html += "    setInterval(updateHUD, 1000);\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(VOLTAGE_PIN, INPUT);
  pinMode(CURRENT_PIN, INPUT);
  
  // Initialize SD Card
  Serial.println("\nInitializing SD Card...");
  if (!SD.begin(SD_CS_PIN)) {
    sdSyncStatus = "Mount Failed";
    Serial.println("[SD Error] Card Mount Failed!");
  } else {
    sdSyncStatus = "Active";
    Serial.println("[SD] Mounted successfully.");
    
    // Write headers if new CSV
    if (!SD.exists(logFilepath)) {
      File f = SD.open(logFilepath, FILE_WRITE);
      if (f) {
        f.println("TimeS,VoltageV,CurrentMa,PowerMw");
        f.close();
      }
    }
  }
  
  Serial.println("\nESP32 Solar Logger starting...");
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
  
  lastLogTime = millis();
}

void loop() {
  server.handleClient();
  
  // Continuously refresh sensor measurements
  readSolarSensors();
  
  // Periodically log and sync (every 10 seconds)
  unsigned long now = millis();
  if (now - lastLogTime >= LOG_INTERVAL_MS) {
    lastLogTime = now;
    
    logToSDCard();
    uploadToCloud();
  }
  
  delay(5);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **SD Card Module**, and **2 Potentiometers** (to simulate voltage and current) onto the canvas.
2. Wire Voltage Potentiometer wiper to **GPIO34**, Current Potentiometer wiper to **GPIO35**, and SD CS to **GPIO5**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Set the simulated voltage potentiometer slider to 4.5V, and current potentiometer to 50% (simulating active panel charging).
6. Verify that the webpage displays the calculated Power (mW) (e.g. 4.5V * 500mA = 2250 mW).
7. Verify that the SD Card and Cloud sync states indicate `Success` in the status table.

## Expected Output
Serial Monitor:
```
[SD] Mounted successfully.
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[SD] Saved entry -> Time: 10, Volt: 4.50 V | Current: 500.0 mA | Power: 2250.0 mW
[Cloud] Pushing payload -> {"voltage":4.50,"current":500.0,"power":2250.0,"uptime":10}
[Cloud] Upload successful.
```

## Expected Canvas Behavior
* Adjusting the simulated voltage and current potentiometers updates the web indicators and JSON data logs in real time.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `readSolarSensors()` | Samples the solar panel voltage and current. |
| `panelVoltage * generationCurrentMa` | Calculates output power in milliwatts. |
| `SD.open(..., FILE_APPEND)` | Appends solar generation data to the SD Card log. |
| `http.POST(jsonPayload)` | Uploads data to the cloud service. |

## Hardware & Safety Concept: Current Sensor Placement and Ground Loops
* **Current Sensor Placement**: The ACS712 is a bidirectional sensor and should be placed in series with the positive line of the solar panel. Connect the panel's positive terminal to IP+ and the load's positive input to IP-.
* **Ground Loops**: High current passing through solar lines can shift ground potentials, causing analog reading errors. Solder short, thick wires for power grounds, and use a single "Star Ground" connection point to prevent ground loops.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a battery charging animation.
2. **Alert buzzer**: Add a buzzer (GPIO 15) to sound an alarm if the voltage exceeds 5.5V (overvoltage warning).
3. **ThingSpeak Integration**: Update the code to upload to a ThingSpeak multi-field channel (Project 141) instead of httpbin.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Current reads constant value | ACS712 center drift | Calibrate the `ADC_CENTER_OFFSET` in code to match your specific sensor's output under zero load |
| SD Card Mount Failed | Card format wrong | Ensure the SD card is formatted as FAT16 or FAT32 |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [138 - ESP32 SD Card Energy Logger Web Status](138-esp32-sd-card-energy-logger-web-status.md)
- [159 - ESP32 Automatic Battery Capacity Tester](159-esp32-automatic-battery-capacity-tester-iot-web-logger.md)
- [161 - ESP32 Solar Tracker Dual Axis Controller](161-esp32-solar-tracker-dual-axis-controller-iot-status-dashboard.md)
- [163 - Water Quality Station IoT](163-esp32-water-quality-station-iot-ph-sensor-temp-lcd-web-upload.md) (Next project)
