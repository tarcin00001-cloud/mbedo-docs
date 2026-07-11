# 139 - IoT Weather Station Node (DHT22 + BMP180 + Cloud database upload)

Build a web-connected environmental weather station node on the ESP32 that samples a DHT22 sensor on GPIO 14, a BMP180 barometric pressure sensor on I2C (GPIO 21/22), hosts a local web dashboard, and pushes telemetry data to a cloud database endpoint periodically.

## Goal
Learn how to sample multiple climate sensors (DHT22 + BMP180), initialize I2C bus sharing, compile HTTP POST requests, serialize data payloads, and process API server response codes.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A DHT22 sensor is on GPIO 14, and a BMP180 sensor on I2C (GPIO 21/22). Every 15 seconds, the ESP32 samples temperature, humidity, pressure, and altitude. It hosts a local webpage to view these readings. Simultaneously, the ESP32 sends an HTTP POST request containing a JSON payload of the weather data to a cloud database simulator (`http://httpbin.org/post`).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temperature & Humidity Sensor | `dht22` | Yes | Yes |
| BMP180 Barometric Pressure Sensor | `i2c_device` | No | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use the simulated DHT22 widget to adjust temperature and humidity. The I2C pressure reading is simulated using random drift offsets around a standard sea-level reference (1013.25 hPa).

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | OUT (Signal) | GPIO14 | Yellow | Climate sensor signal |
| BMP180 Sensor | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power both sensors from the 3.3V rail. Connect a 10 kΩ pull-up resistor between the DHT22 data pin and the 3.3V rail.

## Code
```cpp
// IoT Weather Station Node (DHT22 + BMP180 + Cloud database upload)
#include <WiFi.h>
#include <WebServer.h>
#include <HTTPClient.h>
#include <Wire.h>
#include <Adafruit_BMP085.h>
#include <DHT.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Cloud DB API endpoint
const char* cloudApiUrl = "http://httpbin.org/post";

const int DHT_PIN = 14;
#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

Adafruit_BMP085 bmp;
WebServer server(80);

// Global telemetry variables
float dhtTemp = 0.0;
float dhtHumid = 0.0;
float bmpPressure = 1013.25;
float bmpAltitude = 0.0;

unsigned long lastUploadTime = 0;
const unsigned long UPLOAD_INTERVAL_MS = 15000; // Upload data every 15 seconds
String cloudSyncStatus = "Idle";

// Read sensor metrics
void readSensors() {
  dhtTemp = dht.readTemperature();
  dhtHumid = dht.readHumidity();
  
  // Read barometric pressure and calculate altitude
  if (bmp.begin()) {
    bmpPressure = (float)bmp.readPressure() / 100.0; // Convert Pa to hPa
    bmpAltitude = bmp.readAltitude();
  } else {
    // Simulated values for workspace environment if module is absent
    static float simPressure = 1013.2;
    simPressure += ((float)random(-5, 6) / 10.0);
    bmpPressure = simPressure;
    bmpAltitude = 120.0;
  }
}

// HTTP POST payload upload to cloud DB API
void uploadTelemetryToCloud() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(cloudApiUrl);
    http.addHeader("Content-Type", "application/json");
    
    // Compile JSON payload
    String jsonPayload = "{\"temperature\":" + String(dhtTemp, 1) + 
                         ",\"humidity\":" + String(dhtHumid, 1) + 
                         ",\"pressure\":" + String(bmpPressure, 1) + 
                         ",\"altitude\":" + String(bmpAltitude, 1) + "}";
                         
    Serial.printf("[Cloud API] Sending POST payload -> %s\n", jsonPayload.c_str());
    int httpResponseCode = http.POST(jsonPayload);
    
    if (httpResponseCode > 0) {
      String response = http.getString();
      cloudSyncStatus = "Success (Code: " + String(httpResponseCode) + ")";
      Serial.printf("[Cloud API] Upload successful. Code: %d\n", httpResponseCode);
    } else {
      cloudSyncStatus = "Failed (Error: " + String(httpResponseCode) + ")";
      Serial.printf("[Cloud API] Error sending POST request: %d\n", httpResponseCode);
    }
    http.end();
  } else {
    cloudSyncStatus = "WiFi Offline";
    Serial.println("[Cloud API] Sync blocked. WiFi offline.");
  }
}

// HTTP API endpoint returning JSON data
void handleGetWeather() {
  String json = "{\"temp\":" + String(dhtTemp, 1) + 
                 ",\"humidity\":" + String(dhtHumid, 1) + 
                 ",\"pressure\":" + String(bmpPressure, 1) + 
                 ",\"altitude\":" + String(bmpAltitude, 1) + 
                 ",\"sync\":\"" + cloudSyncStatus + "\"}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Weather Node Dashboard</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 480px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 25px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; text-align: left; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 24px; font-weight: bold; margin-top: 5px; color: #10b981; font-family: monospace; }\n";
  html += "  .badge { display: inline-block; padding: 6px 12px; border-radius: 12px; font-size: 11px; font-weight: bold; text-transform: uppercase; margin-bottom: 10px; background-color: #334155; color: #94a3b8; }\n";
  html += "  .badge.synced { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>IoT Weather Station</h1>\n";
  html += "  <div id=\"syncBadge\" class=\"badge\">Idle</div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Temperature</div><div class=\"metric-val\" id=\"tempDisplay\">--.- &deg;C</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Humidity</div><div class=\"metric-val\" id=\"humidDisplay\">--.- %</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Pressure</div><div class=\"metric-val\" id=\"pressDisplay\">----.- hPa</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Altitude</div><div class=\"metric-val\" id=\"altDisplay\">---.- m</div></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Cloud target: httpbin.org/post | Upload: 15s</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateWeather() {\n";
  html += "    fetch('/api/weather')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('tempDisplay').innerHTML = data.temp.toFixed(1) + ' &deg;C';\n";
  html += "        document.getElementById('humidDisplay').innerText = data.humidity.toFixed(1) + ' %';\n";
  html += "        document.getElementById('pressDisplay').innerText = data.pressure.toFixed(1) + ' hPa';\n";
  html += "        document.getElementById('altDisplay').innerText = data.altitude.toFixed(1) + ' m';\n";
  
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
  html += "    updateWeather();\n";
  html += "    setInterval(updateWeather, 2000); // Poll once every 2 seconds\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  dht.begin();
  
  // Initialize BMP180 sensor
  if (!bmp.begin()) {
    Serial.println("[BMP Error] BMP180 sensor missing! Running in simulation mode.");
  }
  
  Serial.println("\nESP32 Weather Node Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/weather", handleGetWeather);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // Refresh sensor readings
  readSensors();
  
  // Periodically upload data to the cloud
  unsigned long now = millis();
  if (now - lastUploadTime >= UPLOAD_INTERVAL_MS) {
    lastUploadTime = now;
    uploadTelemetryToCloud();
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **DHT22 Sensor** onto the canvas.
2. Wire the DHT22 output to **GPIO14**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Verify that the webpage displays temperature, humidity, and barometric pressure.
6. Verify that the Serial Monitor outputs `Upload successful` and the sync badge on the webpage updates to green.

## Expected Output
Serial Monitor:
```
[BMP Error] BMP180 sensor missing! Running in simulation mode.
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Cloud API] Sending POST payload -> {"temperature":24.0,"humidity":40.0,"pressure":1013.2,"altitude":120.0}
[Cloud API] Upload successful. Code: 200
```

## Expected Canvas Behavior
* Adjusting the simulated DHT22 sliders updates the web indicators and JSON POST payloads sent to the cloud server simulator.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `dht.readTemperature()` | Samples the temperature in Celsius from the DHT22. |
| `bmp.readPressure()` | Samples the barometric pressure from the BMP180. |
| `http.POST(jsonPayload)` | Sends the serialized JSON payload over HTTP. |
| `http.getString()` | Retrieves the response payload from the API server. |

## Hardware & Safety Concept: Sensor Fusion and Power Saving
* **Sensor Fusion**: Weather stations often combine different sensors (like DHT22 for humidity, BMP180 for barometric pressure) to get more accurate data. This is called sensor fusion.
* **Deep Sleep Mode**: Pushing data to the cloud continuously consumes significant battery power. For solar-powered nodes, configure the ESP32 to boot, take sensor readings, upload the payload to the cloud, and enter **Deep Sleep** (`esp_deep_sleep_start()`) for 15 minutes to maximize battery life.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a weather icon (sun/cloud/rain) based on the humidity level.
2. **Alert sound**: Add a buzzer (GPIO 15) to sound an alarm if the pressure drops rapidly (storm warning).
3. **ThingSpeak Integration**: Update the API URL to point to a real ThingSpeak channel (Project 140) to plot data over time.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Upload fails with negative error code | DNS or network issue | Confirm that the ESP32 can resolve the domain `httpbin.org` |
| Pressure readings stay constant | BMP180 missing | Verify the I2C bus wiring (GPIO 21/22) for the BMP180 module |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [36 - ESP32 OLED Digital Clock NTP synced](../beginner/36-esp32-oled-digital-clock-ntp-synced.md)
- [137 - ESP32 SD Card Temp Logger Web Graph](137-esp32-sd-card-temp-logger-web-graph.md)
- [140 - ESP32 ThingSpeak Environment Monitor](140-esp32-thingspeak-environment-monitor-upload-temp-humidity-to-thingspeak-channel.md) (Next project)
