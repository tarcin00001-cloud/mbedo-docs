# 150 - Dual-protocol WiFi-to-SD logger

Build a high-reliability environmental telemetry logger on the ESP32 that samples a DHT22 sensor on GPIO 14, logs readings locally to an SD card over SPI (CS on GPIO 5), and simultaneously uploads data packets to a cloud REST endpoint via HTTP POST.

## Goal
Learn how to implement dual-protocol logging structures (local storage fallback + cloud databases), manage SPI interfaces, compile HTTP REST payloads, and serve diagnostics.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A DHT22 sensor is on GPIO 14, and an SD card module on SPI. Every 15 seconds, the ESP32 reads temperature and humidity. It appends the data to `/log.csv` on the SD card, and publishes a JSON payload of the reading to a cloud database simulator (`http://httpbin.org/post`). A local webpage displays live values, SD card status, and cloud sync logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temperature & Humidity Sensor | `dht22` | Yes | Yes |
| Micro SD Card Reader SPI Module | `sd_card` | Yes | Yes |
| Micro SD Card (FAT16/FAT32 formatted) | `sd_card` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | OUT (Signal) | GPIO14 | Yellow | Climate sensor signal |
| SD Card Module | CS (Chip Select) | GPIO5 | Yellow | SPI chip select |
| SD Card Module | SCK (Clock) | GPIO18 | Green | SPI clock output |
| SD Card Module | MISO (Data Out)| GPIO19 | Blue | SPI master input |
| SD Card Module | MOSI (Data In) | GPIO23 | White | SPI master output |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power both the DHT22 sensor and the SD Card module from the 5V Vin rail.

## Code
```cpp
// Dual-protocol WiFi-to-SD logger (DHT22 sensor + SPI SD card + Cloud REST Client)
#include <WiFi.h>
#include <WebServer.h>
#include <HTTPClient.h>
#include <SPI.h>
#include <SD.h>
#include <DHT.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Cloud DB API Endpoint
const char* cloudApiUrl = "http://httpbin.org/post";

const int DHT_PIN = 14;
#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

const int SD_CS_PIN = 5;
const char* logFilepath = "/log.csv";

WebServer server(80);

// Global values
float currentTemp = 0.0;
float currentHumid = 0.0;

unsigned long lastLogTime = 0;
const unsigned long LOG_INTERVAL_MS = 15000; // Log data every 15 seconds

String sdSyncStatus = "Not Initialized";
String cloudSyncStatus = "Idle";

// Protocol 1: Write data locally to SD card CSV file
bool logToSDCard(float t, float h) {
  File file = SD.open(logFilepath, FILE_APPEND);
  if (file) {
    unsigned long timestampS = millis() / 1000;
    file.printf("%lu,%.1f,%.1f\n", timestampS, t, h);
    file.close();
    sdSyncStatus = "Success (Logged)";
    Serial.printf("[SD] Saved entry -> Time: %lu, Temp: %.1f C, Humid: %.1f%%\n", timestampS, t, h);
    return true;
  } else {
    sdSyncStatus = "Write Error";
    Serial.println("[SD Error] Failed to open CSV file for appending!");
    return false;
  }
}

// Protocol 2: Upload data packet to cloud REST API
bool uploadToCloud(float t, float h) {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(cloudApiUrl);
    http.addHeader("Content-Type", "application/json");
    
    // Compile JSON payload
    String jsonPayload = "{\"temperature\":" + String(t, 1) + 
                         ",\"humidity\":" + String(h, 1) + 
                         ",\"timestamp\":" + String(millis() / 1000) + "}";
                         
    Serial.printf("[Cloud] Pushing payload -> %s\n", jsonPayload.c_str());
    int httpResponseCode = http.POST(jsonPayload);
    http.end();
    
    if (httpResponseCode == 200 || httpResponseCode == 201) {
      cloudSyncStatus = "Success (Code: " + String(httpResponseCode) + ")";
      Serial.println("[Cloud] Upload successful.");
      return true;
    } else {
      cloudSyncStatus = "Failed (Code: " + String(httpResponseCode) + ")";
      Serial.printf("[Cloud Error] POST failed: %d\n", httpResponseCode);
      return false;
    }
  } else {
    cloudSyncStatus = "WiFi Offline";
    Serial.println("[Cloud] Upload blocked. WiFi offline.");
    return false;
  }
}

// HTTP API endpoint returning JSON data
void handleGetStatus() {
  String json = "{\"temp\":" + String(currentTemp, 1) + 
                 ",\"humidity\":" + String(currentHumid, 1) + 
                 ",\"sd\":\"" + sdSyncStatus + "\"" +
                 ",\"cloud\":\"" + cloudSyncStatus + "\"}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Dual-Protocol Logger</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 480px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 25px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; text-align: left; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 24px; font-weight: bold; margin-top: 5px; color: #10b981; font-family: monospace; }\n";
  html += "  .sync-table { width: 100%; border-collapse: collapse; margin-top: 20px; text-align: left; font-size: 13px; }\n";
  html += "  .sync-table th { color: #64748b; border-bottom: 2px solid #334155; padding-bottom: 8px; }\n";
  html += "  .sync-table td { padding: 10px 0; border-bottom: 1px solid #1e293b; color: #f1f5f9; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Dual-Protocol Data Logger</h1>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Temperature</div><div class=\"metric-val\" id=\"tempDisplay\">--.- &deg;C</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Humidity</div><div class=\"metric-val\" id=\"humidDisplay\">--.- %</div></div>\n";
  html += "  </div>\n";
  
  html += "  <table class=\"sync-table\">\n";
  html += "    <thead><tr><th>Protocol Channel</th><th>Synchronization Status</th></tr></thead>\n";
  html += "    <tbody>\n";
  html += "      <tr><td>Local SD Card</td><td id=\"sdStatus\">Not Initialized</td></tr>\n";
  html += "      <tr><td>Cloud REST API</td><td id=\"cloudStatus\">Idle</td></tr>\n";
  html += "    </tbody>\n";
  html += "  </table>\n";
  
  html += "  <p class=\"footer\">SD Card: CS pin 5 | Cloud: httpbin.org | Interval: 15s</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateHUD() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('tempDisplay').innerHTML = data.temp.toFixed(1) + ' &deg;C';\n";
  html += "        document.getElementById('humidDisplay').innerText = data.humidity.toFixed(1) + ' %';\n";
  html += "        document.getElementById('sdStatus').innerText = data.sd;\n";
  html += "        document.getElementById('cloudStatus').innerText = data.cloud;\n";
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
  
  dht.begin();
  
  // 1. Initialize SD Card
  Serial.println("\nInitializing SD Card...");
  if (!SD.begin(SD_CS_PIN)) {
    sdSyncStatus = "SD Mount Failed";
    Serial.println("[SD Error] Card Mount Failed!");
  } else {
    sdSyncStatus = "Active";
    Serial.println("[SD] Mounted successfully.");
    
    // Initialize file header if new file
    if (!SD.exists(logFilepath)) {
      File f = SD.open(logFilepath, FILE_WRITE);
      if (f) {
        f.println("TimeS,TempC,Humidity");
        f.close();
      }
    }
  }
  
  Serial.println("\nESP32 Dual-Protocol node starting...");
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
  
  // Refresh environmental variables
  currentTemp = dht.readTemperature();
  currentHumid = dht.readHumidity();
  
  // Periodically log data (every 15 seconds)
  unsigned long now = millis();
  if (now - lastLogTime >= LOG_INTERVAL_MS) {
    lastLogTime = now;
    
    if (!isnan(currentTemp) && !isnan(currentHumid)) {
      // Execute both channels
      logToSDCard(currentTemp, currentHumid);
      uploadToCloud(currentTemp, currentHumid);
    }
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22 Sensor**, and **SD Card Module** onto the canvas.
2. Wire DHT22 output to **GPIO14**, and SD CS to **GPIO5**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Verify that the webpage lists "Local SD Card: Active" and "Cloud REST API: Idle".
6. Wait 20 seconds. Verify that both channels update to "Success" and the Serial Monitor prints the logs.

## Expected Output
Serial Monitor:
```
[SD] Mounted successfully.
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[SD] Saved entry -> Time: 15, Temp: 24.0 C, Humid: 40.0%
[Cloud] Pushing payload -> {"temperature":24.0,"humidity":40.0,"timestamp":15}
[Cloud] Upload successful.
```

## Expected Canvas Behavior
* Adjusting the simulated DHT22 sliders writes logs to the SD Card image and sends JSON packets to the cloud database simultaneously.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `logToSDCard(currentTemp, ...)` | Channel 1: Appends the data locally to `/log.csv`. |
| `uploadToCloud(currentTemp, ...)` | Channel 2: Uploads the JSON packet to the REST API. |
| `WiFi.status() == WL_CONNECTED` | Checks if the network is available before executing HTTP requests. |

## Hardware & Safety Concept: Network Failures and Offline Queue Syncing
* **Offline Queue**: During network outages, the cloud channel will fail, but the local SD card continues logging. A resilient IoT system should flag the failed cloud logs and, once WiFi reconnects, read the missing records from the SD card and upload them in a batch to synchronize the cloud database.
* **Corrupt File recovery**: If an SD card is removed or power is cut during a write, the file system can corrupt. Check the return code of `SD.begin()` periodically. If it fails, try re-initializing the card to recover automatically.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display separate checkmarks for SD Card and Cloud sync states.
2. **Alert sound**: Add a buzzer (GPIO 15) to chirp if either channel fails.
3. **ThingSpeak Dual-channel**: Update the code to upload to ThingSpeak (Project 140) and write to the SD card.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| SD Card shows "Mount Failed" | Card format issue | Ensure the virtual or physical SD card is formatted as FAT16 or FAT32 |
| Cloud upload fails but SD works | Wi-Fi offline | Verify that the ESP32 is connected to Wi-Fi and the router has internet access |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [136 - ESP32 SD Card Data Logger Web Viewer](136-esp32-sd-card-data-logger-web-viewer-host-directory-listing.md)
- [140 - ESP32 ThingSpeak Environment Monitor](140-esp32-thingspeak-environment-monitor-upload-temp-humidity-to-thingspeak-channel.md)
- [151 - Web Page PID Parameter Tuner](151-esp32-web-page-pid-parameter-tuner.md) (Next project)
