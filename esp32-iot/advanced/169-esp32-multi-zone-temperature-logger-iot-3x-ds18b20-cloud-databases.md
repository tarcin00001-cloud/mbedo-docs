# 169 - Multi-zone Temperature logger IoT (3x DS18B20 -> Cloud databases)

Build an internet-connected multi-zone temperature telemetry node on the ESP32 that reads three DS18B20 temperature probes connected to a single OneWire data bus on GPIO 12, displays measurements locally on an I2C LCD 1602 (GPIO 21/22), and uploads the multi-zone readings to a cloud database using REST API POST requests.

## Goal
Learn how to address multiple OneWire sensors on a single shared bus line, index individual temperature probes, write status pages to I2C LCDs, and upload multi-field JSON payloads to REST APIs.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Three DS18B20 sensors are wired in parallel to GPIO 12. The ESP32 requests conversions and reads temperatures by index (Zone A, Zone B, Zone C). The values are displayed on an I2C LCD 1602. Every 10 seconds, the ESP32 uploads a JSON payload containing the three zone temperatures to a cloud API (`http://httpbin.org/post`). A local webpage displays live values and sync logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DS18B20 Waterproof Probes | `ds18b20` | Yes (3 pieces) | Yes (3 pieces) |
| I2C LCD 1602 Display | `lcd_i2c` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS18B20 (Zone A) | DQ (Data) | GPIO12 | Green | Parallel OneWire bus DQ |
| DS18B20 (Zone B) | DQ (Data) | GPIO12 | Green | Parallel OneWire bus DQ |
| DS18B20 (Zone C) | DQ (Data) | GPIO12 | Green | Parallel OneWire bus DQ |
| LCD 1602 Display| SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Solder a single 4.7 kΩ pull-up resistor between the shared OneWire data line (GPIO 12) and the 3.3V power line.

## Code
```cpp
// Multi-zone Temperature logger IoT (3x DS18B20 on single pin + I2C LCD + Cloud REST upload)
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

const int ONE_WIRE_BUS = 12;

OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

LiquidCrystal_I2C lcd(0x27, 16, 2);
WebServer server(80);

// Global zone temperatures
float tempZoneA = 22.0;
float tempZoneB = 22.0;
float tempZoneC = 22.0;

unsigned long lastUploadTime = 0;
const unsigned long UPLOAD_INTERVAL_MS = 10000; // Upload every 10 seconds

String cloudSyncStatus = "Idle";

// Read all 3 sensors and display on local LCD
void readZoneSensors() {
  // 1. Request conversions across the entire OneWire bus
  sensors.requestTemperatures();
  
  // Read temperatures by index
  float tA = sensors.getTempCByIndex(0);
  float tB = sensors.getTempCByIndex(1);
  float tC = sensors.getTempCByIndex(2);
  
  // Update state variables (checking for disconnected probes)
  if (tA > -127.0) tempZoneA = tA;
  if (tB > -127.0) tempZoneB = tB;
  if (tC > -127.0) tempZoneC = tC;
  
  // Mock simulation values in MbedO if probes are not present
  if (tA <= -127.0 && tB <= -127.0) {
    static float simA = 21.5, simB = 23.8, simC = 19.2;
    simA += ((float)random(-1, 2) / 10.0);
    simB += ((float)random(-1, 2) / 10.0);
    simC += ((float)random(-1, 2) / 10.0);
    tempZoneA = simA;
    tempZoneB = simB;
    tempZoneC = simC;
  }
  
  // 2. Display on I2C LCD (Alternating pages every loop)
  static bool alternatePage = false;
  lcd.clear();
  if (!alternatePage) {
    lcd.setCursor(0, 0);
    lcd.print("Zone A: ");
    lcd.print(tempZoneA, 1);
    lcd.print(" C");
    lcd.setCursor(0, 1);
    lcd.print("Zone B: ");
    lcd.print(tempZoneB, 1);
    lcd.print(" C");
  } else {
    lcd.setCursor(0, 0);
    lcd.print("Zone C: ");
    lcd.print(tempZoneC, 1);
    lcd.print(" C");
    lcd.setCursor(0, 1);
    lcd.print("WiFi Connected");
  }
  alternatePage = !alternatePage;
}

// Upload multi-zone data package to cloud REST API
void uploadTelemetryToCloud() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(cloudApiUrl);
    http.addHeader("Content-Type", "application/json");
    
    // Compile JSON payload containing all three zones
    String jsonPayload = "{\"zoneA\":" + String(tempZoneA, 2) + 
                         ",\"zoneB\":" + String(tempZoneB, 2) + 
                         ",\"zoneC\":" + String(tempZoneC, 2) + 
                         ",\"uptime\":" + String(millis() / 1000) + "}";
                         
    Serial.printf("[Cloud Bridge] Pushing payload -> %s\n", jsonPayload.c_str());
    int httpResponseCode = http.POST(jsonPayload);
    
    if (httpResponseCode == 200 || httpResponseCode == 201) {
      cloudSyncStatus = "Success (Code: " + String(httpResponseCode) + ")";
      Serial.println("[Cloud Bridge] Upload successful.");
    } else {
      cloudSyncStatus = "Failed (Code: " + String(httpResponseCode) + ")";
      Serial.printf("[Cloud Bridge Error] POST failed: %d\n", httpResponseCode);
    }
    http.end();
  } else {
    cloudSyncStatus = "WiFi Offline";
  }
}

// HTTP API endpoint returning JSON data
void handleGetStatus() {
  String json = "{\"zoneA\":" + String(tempZoneA, 1) + 
                 ",\"zoneB\":" + String(tempZoneB, 1) + 
                 ",\"zoneC\":" + String(tempZoneC, 1) + 
                 ",\"sync\":\"" + cloudSyncStatus + "\"}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Multi-Zone Climate Monitor</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 500px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .badge { display: inline-block; padding: 6px 12px; border-radius: 12px; font-size: 11px; font-weight: bold; text-transform: uppercase; margin-bottom: 20px; background-color: #334155; color: #94a3b8; }\n";
  html += "  .badge.synced { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .zones-grid { display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 10px; margin: 20px 0; }\n";
  html += "  .zone-card { background-color: #0f172a; border-radius: 8px; padding: 12px; border: 1px solid #334155; text-align: left; }\n";
  html += "  .zone-label { font-size: 9px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .zone-val { font-size: 18px; font-weight: bold; margin-top: 5px; color: #10b981; font-family: monospace; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Multi-Zone Telemetry</h1>\n";
  html += "  <div id=\"syncBadge\" class=\"badge\">Idle</div>\n";
  
  // Zone grids
  html += "  <div class=\"zones-grid\">\n";
  html += "    <div class=\"zone-card\"><span class=\"zone-label\">Zone A</span><div class=\"zone-val\" id=\"zA\">0.0&deg;C</div></div>\n";
  html += "    <div class=\"zone-card\"><span class=\"zone-label\">Zone B</span><div class=\"zone-val\" id=\"zB\">0.0&deg;C</div></div>\n";
  html += "    <div class=\"zone-card\"><span class=\"zone-label\">Zone C</span><div class=\"zone-val\" id=\"zC\">0.0&deg;C</div></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">OneWire Pin: 12 | Target: httpbin.org | Upload: 10s</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateHUD() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('zA').innerHTML = data.zoneA.toFixed(1) + ' &deg;C';\n";
  html += "        document.getElementById('zB').innerHTML = data.zoneB.toFixed(1) + ' &deg;C';\n";
  html += "        document.getElementById('zC').innerHTML = data.zoneC.toFixed(1) + ' &deg;C';\n";
  
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
  html += "    updateHUD();\n";
  html += "    setInterval(updateHUD, 2000);\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  sensors.begin();
  
  // Initialize I2C LCD
  lcd.init();
  lcd.backlight();
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Zone Logger");
  lcd.setCursor(0, 1);
  lcd.print("Init OneWire...");
  
  Serial.println("\nESP32 Multi-Zone Temp Logger starting...");
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
  
  lastUploadTime = millis();
}

void loop() {
  server.handleClient();
  
  // Periodically refresh sensor calculations (every 5 seconds)
  static unsigned long lastSampleTime = 0;
  if (millis() - lastSampleTime >= 5000) {
    lastSampleTime = millis();
    readZoneSensors();
  }
  
  // Periodically upload data package to cloud REST API (every 10 seconds)
  unsigned long now = millis();
  if (now - lastUploadTime >= UPLOAD_INTERVAL_MS) {
    lastUploadTime = now;
    uploadTelemetryToCloud();
  }
  
  delay(5);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **I2C LCD 1602**, and **3 DS18B20 Temperature Probes** onto the canvas.
2. Wire the DQ (Data) lines of all three DS18B20 sensors in parallel to **GPIO12**.
3. Wire the SDA and SCL lines of the LCD display to **GPIO21 and GPIO22**.
4. Paste the code and click **Run**.
5. Open your web browser and navigate to the printed IP address.
6. Set the sliders of the three simulated DS18B20 temperature probes to different temperatures (e.g. 21.0 °C, 24.5 °C, and 18.0 °C).
7. Verify that the local LCD character display alternates every 5 seconds to show the current values for each zone.
8. Verify that the webpage displays the correct temperature readings for all three zones, and the cloud sync indicator updates to show `Success`.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Cloud Bridge] Pushing payload -> {"zoneA":21.0,"zoneB":24.5,"zoneC":18.0,"uptime":10}
[Cloud Bridge] Upload successful.
```

## Expected Canvas Behavior
* Adjusting any of the three simulated DS18B20 probe sliders updates the alternating character text on the I2C LCD and the webpage cards.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `sensors.requestTemperatures()` | Commands all sensors connected to the OneWire bus to measure temperature. |
| `sensors.getTempCByIndex(0)` | Reads the temperature value from the first sensor addressed on the bus. |
| `alternatePage = !alternatePage` | Toggles the active screen page printed on the I2C LCD. |
| `http.POST(jsonPayload)` | Initiates the HTTP POST request to upload the multi-zone data package. |

## Hardware & Safety Concept: OneWire ROM Addressing and Bus Limits
* **OneWire ROM Addressing**: Every DS18B20 sensor has a unique 64-bit ROM code burned into its chip at the factory. The Dallas library allows you to request temperatures by index (`0`, `1`, `2`). However, if a sensor is disconnected, the indices of the remaining sensors will shift, corrupting your zone mapping. In production systems, discover the unique ROM codes of your sensors beforehand and read them directly using their addresses: `sensors.getTempC(deviceAddress)`.
* **Bus Limits**: You can connect dozens of DS18B20 sensors to a single GPIO pin, but a single 4.7 kΩ pull-up resistor is only sufficient for up to 5–10 sensors on short wires. For longer cables or more sensors, decrease the pull-up resistance to 2.2 kΩ to ensure clean signals.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a 3-channel scrolling line graph of all three zones.
2. **Freeze Alert Relay**: Add a relay (GPIO 2) to activate a heater if any of the zone temperatures drop below 5.0 °C.
3. **SPIFFS integration**: Save daily temperature logs to a CSV file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD display stays blank | Contrast potentiometer | Adjust the contrast potentiometer on the back of the I2C backpack board |
| Temperatures read -127 °C | Pull-up resistor missing | Ensure a 4.7 kΩ pull-up resistor is connected between the shared DQ line (GPIO 12) and the 3.3V power rail |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [100 - Smart Fan (MbedO Curriculum)](100-smart-fan.md)
- [122 - Cold Chain Monitor Node](../expert/122-cold-chain-monitor.md)
- [163 - Water Quality Station IoT](163-esp32-water-quality-station-iot-ph-sensor-temp-lcd-web-upload.md)
- [170 - NeoPixel Web Color Wheel picker](170-esp32-neopixel-web-color-wheel-picker.md) (Next project)
