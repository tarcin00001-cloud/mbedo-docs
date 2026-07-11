# 138 - SD Card Energy Logger Web status

Build an automated smart energy logger on the ESP32 that samples an ACS712 current sensor on GPIO 34, calculates active current, power, and accumulated energy (Watt-hours), appends logs to an SD card, and hosts a web dashboard with live statistics.

## Goal
Learn how to sample current sensors, calculate power and energy metrics, log data to SD cards, serve web dashboards, and handle reset override requests.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An ACS712 current sensor is on GPIO 34, and an SD card module on SPI. The ESP32 measures current consumption. It calculates active current, power (Watts), and accumulated energy (Wh), appending logs to `/energy.csv` on the SD card. Navigating to the ESP32's IP address displays a dashboard showing live power statistics and recent logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| ACS712 Current Sensor Module | `potentiometer` | Yes | Yes |
| Micro SD Card Reader SPI Module | `sd_card` | Yes | Yes |
| Micro SD Card (FAT16/FAT32 formatted) | `sd_card` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use a potentiometer to simulate the analog output of the ACS712 current sensor.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| ACS712 Sensor | OUT (Analog) | GPIO34 | Yellow | Current signal input |
| SD Card Module | CS (Chip Select) | GPIO5 | Yellow | SPI chip select |
| SD Card Module | SCK (Clock) | GPIO18 | Green | SPI clock output |
| SD Card Module | MISO (Data Out)| GPIO19 | Blue | SPI master input |
| SD Card Module | MOSI (Data In) | GPIO23 | White | SPI master output |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the ACS712 sensor and SD Card module from the 5V Vin rail.

## Code
```cpp
// SD Card Energy Logger Web status (ACS712 current mapping + Energy accumulator + SPI SD logger)
#include <WiFi.h>
#include <WebServer.h>
#include <SPI.h>
#include <SD.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int ACS712_PIN = 34;
const int SD_CS_PIN = 5;
const char* logFilepath = "/energy.csv";

WebServer server(80);

// Calibration offsets
const int ADC_CENTER_OFFSET = 2048; // Vcc/2 reference for 12-bit ADC (2.5V)
const double SENSITIVITY_V_PER_A = 0.185; // 185 mV/A for 5A model ACS712
const double VOLTAGE_RMS = 230.0; // Standard mains voltage representation

double currentAmps = 0.0;
double powerWatts = 0.0;
double totalEnergyWh = 0.0;

unsigned long lastSampleTime = 0;
unsigned long lastLogTime = 0;
const unsigned long LOG_INTERVAL_MS = 10000; // Log data every 10 seconds

// Read current sensor and update calculations
void calculateEnergyMetrics() {
  unsigned long now = millis();
  double elapsedSec = (double)(now - lastSampleTime) / 1000.0;
  lastSampleTime = now;
  
  // Sample current
  int rawVal = analogRead(ACS712_PIN);
  
  // Calculate voltage offset (0 to 3.3V range)
  double voltageDiff = ((double)(rawVal - ADC_CENTER_OFFSET) / 4095.0) * 3.3;
  
  // Convert voltage difference to current in Amps
  currentAmps = abs(voltageDiff / SENSITIVITY_V_PER_A);
  
  // Filter out tiny noise fluctuations
  if (currentAmps < 0.05) currentAmps = 0.0;
  
  // Calculate Power (Watts) and Energy (Watt-hours)
  powerWatts = currentAmps * VOLTAGE_RMS;
  totalEnergyWh += (powerWatts * elapsedSec) / 3600.0; // Wh accumulator
}

// HTTP API endpoint returning JSON data
void handleGetEnergyData() {
  String json = "{\"amps\":" + String(currentAmps, 2) + 
                 ",\"watts\":" + String(powerWatts, 1) + 
                 ",\"energyWh\":" + String(totalEnergyWh, 3) + "}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to reset energy counter
void handlePostReset() {
  totalEnergyWh = 0.0;
  Serial.println("[Energy] Accumulator reset to zero.");
  
  // Log reset event
  File file = SD.open(logFilepath, FILE_APPEND);
  if (file) {
    file.println("RESET,0.0,0.0,0.0");
    file.close();
  }
  
  server.send(200, "text/plain", "OK");
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Smart Energy Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .power-display { font-size: 56px; font-weight: 800; font-family: monospace; color: #10b981; margin: 15px 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 20px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 22px; font-weight: bold; margin-top: 5px; color: #38bdf8; font-family: monospace; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 14px; font-size: 16px; font-weight: bold; color: white; background-color: #ef4444; border: none; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; margin-top: 15px; }\n";
  html += "  .btn:hover { background-color: #dc2626; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Smart Energy Logger</h1>\n";
  html += "  <div id=\"wattsDisplay\" class=\"power-display\">0.0 W</div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Current</div><div class=\"metric-val\" id=\"ampsDisplay\">0.00 A</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Energy Used</div><div class=\"metric-val\" id=\"whDisplay\">0.000 Wh</div></div>\n";
  html += "  </div>\n";
  
  html += "  <button class=\"btn\" onclick=\"resetEnergy()\">Reset Accumulator</button>\n";
  html += "  <p class=\"footer\">Data logged to SD card /energy.csv | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateHUD() {\n";
  html += "    fetch('/api/energy')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('wattsDisplay').innerText = data.watts.toFixed(1) + ' W';\n";
  html += "        document.getElementById('ampsDisplay').innerText = data.amps.toFixed(2) + ' A';\n";
  html += "        document.getElementById('whDisplay').innerText = data.energyWh.toFixed(3) + ' Wh';\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  function resetEnergy() {\n";
  html += "    fetch('/api/reset', { method: 'POST' })\n";
  html += "      .then(() => updateHUD());\n";
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
  
  pinMode(ACS712_PIN, INPUT);
  
  Serial.println("\nInitializing SD Card...");
  if (!SD.begin(SD_CS_PIN)) {
    Serial.println("[SD Error] Card Mount Failed! System halted.");
    for (;;);
  }
  Serial.println("[SD] Mounted successfully.");
  
  // Initialize CSV file with headers
  if (!SD.exists(logFilepath)) {
    File f = SD.open(logFilepath, FILE_WRITE);
    if (f) {
      f.println("TimeS,CurrentA,PowerW,EnergyWh");
      f.close();
    }
  }
  
  Serial.println("\nESP32 Energy Logger starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/energy", HTTP_GET, handleGetEnergyData);
  server.on("/api/reset", HTTP_POST, handlePostReset);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  lastSampleTime = millis();
  lastLogTime = millis();
}

void loop() {
  server.handleClient();
  
  // Update calculations continuously
  calculateEnergyMetrics();
  
  // Log metrics to SD card periodically (every 10 seconds)
  unsigned long now = millis();
  if (now - lastLogTime >= LOG_INTERVAL_MS) {
    lastLogTime = now;
    
    File file = SD.open(logFilepath, FILE_APPEND);
    if (file) {
      unsigned long timestampS = now / 1000;
      file.printf("%lu,%.2f,%.1f,%.4f\n", timestampS, currentAmps, powerWatts, totalEnergyWh);
      file.close();
      
      Serial.printf("[SD LOG] Saved data -> Current: %.2f A, Power: %.1f W, Energy: %.3f Wh\n", 
                    currentAmps, powerWatts, totalEnergyWh);
    }
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer** (to simulate the current sensor), and **SD Card Module** onto the canvas.
2. Wire Potentiometer to **GPIO34**, and SD CS to **GPIO5**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. In normal state, keep the potentiometer slider centered (simulating no current). Verify that the readings show 0 W.
6. Slide the potentiometer to 80% (simulating high load current). Verify that the web display shows active current, power (Watts), and accumulating energy (Wh).
7. Click the "Reset Accumulator" button on the webpage. Verify that the Wh value resets to zero.

## Expected Output
Serial Monitor:
```
[SD] Mounted successfully.
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[SD LOG] Saved data -> Current: 0.85 A, Power: 195.5 W, Energy: 0.543 Wh
[Energy] Accumulator reset to zero.
```

## Expected Canvas Behavior
* Adjusting the potentiometer simulates load changes, causing the web indicators to calculate power and Wh metrics.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `analogRead(ACS712_PIN)` | Samples the analog current sensor output voltage. |
| `abs(voltageDiff / SENSITIVITY_V_PER_A)` | Converts voltage difference to RMS current in Amps. |
| `totalEnergyWh += ...` | Accumulates Watt-hours based on active power and elapsed time. |
| `SD.open(..., FILE_APPEND)` | Appends the current energy data row to the log file. |

## Hardware & Safety Concept: Current Sensor Offset and Power Integration
* **Sensor Offset**: The ACS712 current sensor uses a Hall effect sensor to measure magnetic fields generated by current passing through the trace. It outputs a voltage centered at Vcc/2 (2.5V) when no current is flowing. Since the ESP32 uses 3.3V logic, configure a voltage divider or select a compatible 3.3V model (like the ACS712-3.3V) to prevent damage.
* **Mains Safety Warning**: AC current measurements involve high voltages. Never work on live AC mains lines without proper insulation and supervisor oversight. Always use isolation relays and keep high-voltage AC paths physically separated from low-voltage DC logic circuits.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a real-time graph of the current.
2. **Current alert chime**: Add a buzzer (GPIO 15) and sound an alarm if the current exceeds 4.0 Amps.
3. **SPIFFS backup**: Back up the Wh accumulator value to SPIFFS so the total energy use is saved on power loss.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Energy readings drift in normal state | Noise on the analog pin | Increase the noise floor filter value in the code (e.g. from 0.05 to 0.10 Amps) |
| SD Card Mount Failed | Card format wrong | Ensure the SD card is formatted as FAT16 or FAT32 |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [62 - ESP32 SPIFFS file logger](../intermediate/62-esp32-spiffs-file-logger-oled-diagnostics.md)
- [137 - ESP32 SD Card Temp Logger Web Graph](137-esp32-sd-card-temp-logger-web-graph.md)
- [139 - ESP32 IoT Weather Station Node](139-esp32-iot-weather-station-node-dht22-bmp180-cloud-database-upload.md) (Next project)
