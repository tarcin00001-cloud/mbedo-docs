# 140 - ThingSpeak Environment Monitor (Upload temp/humidity to ThingSpeak channel)

Build an environmental telemetry logger on the ESP32 that samples a DHT22 sensor on GPIO 14, uploads temperature and humidity data to a ThingSpeak IoT channel using an HTTP GET request, and hosts a web dashboard with status updates.

## Goal
Learn how to interact with the ThingSpeak cloud database API, compile query parameters in URL strings, handle REST API rate limits, and display connection diagnostics.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A DHT22 sensor is on GPIO 14. Every 20 seconds, the ESP32 reads temperature and humidity. It compiles an HTTP GET query containing its Write API Key and data points, sending it to the ThingSpeak server (`api.thingspeak.com`). Navigating to the ESP32's IP address displays a webpage showing the latest readings and the status of the cloud sync.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temperature & Humidity Sensor | `dht22` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | OUT (Signal) | GPIO14 | Yellow | Climate sensor signal |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the DHT22 from the 3.3V rail. Connect a 10 kΩ pull-up resistor between the data pin and 3.3V rail.

## Code
```cpp
// ThingSpeak Environment Monitor (DHT22 sensor + ThingSpeak API Client)
#include <WiFi.h>
#include <WebServer.h>
#include <HTTPClient.h>
#include <DHT.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// ThingSpeak API configuration
const char* thingSpeakHost = "http://api.thingspeak.com/update";
const String thingSpeakWriteKey = "XYZ123456789"; // Place your ThingSpeak Write API key here

const int DHT_PIN = 14;
#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

WebServer server(80);

float currentTemp = 0.0;
float currentHumid = 0.0;

unsigned long lastUploadTime = 0;
const unsigned long UPLOAD_INTERVAL_MS = 20000; // ThingSpeak limit is 15 seconds
String thingSpeakStatus = "Idle";

// HTTP GET request to update ThingSpeak channel
void uploadToThingSpeak(float t, float h) {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    
    // Construct query parameters URL
    // e.g. http://api.thingspeak.com/update?api_key=XYZ123456789&field1=24.5&field2=50.2
    String url = String(thingSpeakHost) + "?api_key=" + thingSpeakWriteKey + 
                 "&field1=" + String(t, 1) + 
                 "&field2=" + String(h, 1);
                 
    Serial.printf("[ThingSpeak] Request URL -> %s\n", url.c_str());
    http.begin(url);
    
    int httpResponseCode = http.GET();
    
    if (httpResponseCode > 0) {
      String response = http.getString();
      
      // ThingSpeak returns the updated entry ID on success (or 0 if rate limited)
      if (response == "0") {
        thingSpeakStatus = "Rate Limited (Wait 15s)";
        Serial.println("[ThingSpeak] Warning: Upload rate limited!");
      } else {
        thingSpeakStatus = "Success (Entry ID: " + response + ")";
        Serial.printf("[ThingSpeak] Upload successful. Entry ID: %s\n", response.c_str());
      }
    } else {
      thingSpeakStatus = "Failed (Error: " + String(httpResponseCode) + ")";
      Serial.printf("[ThingSpeak] Error sending GET request: %d\n", httpResponseCode);
    }
    http.end();
  } else {
    thingSpeakStatus = "WiFi Offline";
    Serial.println("[ThingSpeak] Sync blocked. WiFi offline.");
  }
}

// HTTP API endpoint returning JSON data
void handleGetTelemetry() {
  String json = "{\"temp\":" + String(currentTemp, 1) + 
                 ",\"humidity\":" + String(currentHumid, 1) + 
                 ",\"sync\":\"" + thingSpeakStatus + "\"}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>ThingSpeak Weather HUD</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 25px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; text-align: left; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 24px; font-weight: bold; margin-top: 5px; color: #10b981; font-family: monospace; }\n";
  html += "  .badge { display: inline-block; padding: 6px 12px; border-radius: 12px; font-size: 11px; font-weight: bold; text-transform: uppercase; margin-bottom: 10px; background-color: #334155; color: #94a3b8; }\n";
  html += "  .badge.synced { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .badge.warning { background-color: #92400e; color: #fef3c7; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>ThingSpeak Weather Node</h1>\n";
  html += "  <div id=\"syncBadge\" class=\"badge\">Idle</div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Temperature</div><div class=\"metric-val\" id=\"tempDisplay\">--.- &deg;C</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Humidity</div><div class=\"metric-val\" id=\"humidDisplay\">--.- %</div></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Cloud Server: api.thingspeak.com | Sync: 20 seconds</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateWeather() {\n";
  html += "    fetch('/api/weather')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('tempDisplay').innerHTML = data.temp.toFixed(1) + ' &deg;C';\n";
  html += "        document.getElementById('humidDisplay').innerText = data.humidity.toFixed(1) + ' %';\n";
  
  html += "        const bdgEl = document.getElementById('syncBadge');\n";
  html += "        bdgEl.innerText = data.sync;\n";
  html += "        if (data.sync.startsWith('Success')) {\n";
  html += "          bdgEl.className = 'badge synced';\n";
  html += "        } else if (data.sync.startsWith('Rate')) {\n";
  html += "          bdgEl.className = 'badge warning';\n";
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
  
  Serial.println("\nESP32 ThingSpeak Client starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/weather", handleGetTelemetry);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // Periodically read sensor and upload to ThingSpeak (every 20 seconds)
  unsigned long now = millis();
  if (now - lastUploadTime >= UPLOAD_INTERVAL_MS) {
    lastUploadTime = now;
    
    currentTemp = dht.readTemperature();
    currentHumid = dht.readHumidity();
    
    if (!isnan(currentTemp) && !isnan(currentHumid)) {
      uploadToThingSpeak(currentTemp, currentHumid);
    } else {
      Serial.println("[DHT Error] Failed to read sensor data!");
    }
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **DHT22 Sensor** onto the canvas.
2. Wire the DHT22 output to **GPIO14**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Verify that the webpage displays temperature and humidity.
6. Verify that the Serial Monitor outputs `Request URL` and the sync badge on the webpage updates to green.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[ThingSpeak] Request URL -> http://api.thingspeak.com/update?api_key=XYZ123456789&field1=24.0&field2=40.0
[ThingSpeak] Upload successful. Entry ID: 154
```

## Expected Canvas Behavior
* Adjusting the simulated DHT22 sliders updates the web indicators and GET query string variables sent to the ThingSpeak cloud simulator.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `dht.readTemperature()` | Samples the temperature in Celsius from the DHT22. |
| `url = ... + "?api_key=" + ...` | Formats the query parameters URL string for ThingSpeak. |
| `http.GET()` | Sends the HTTP GET request. |
| `response == "0"` | Processes the rate-limiting warning response. |

## Hardware & Safety Concept: REST API Rate Limits and Non-blocking Timers
* **Rate Limits**: Free IoT services enforce rate limits to protect their servers. ThingSpeak limits free accounts to one upload every 15 seconds. If you send updates too quickly, the server returns `0` and ignores the data. Enforce an upload interval of 20 seconds to prevent data loss.
* **Non-blocking loops**: Always use `millis()` timers to schedule uploads. Using `delay(20000)` inside the loop freezes the web server, causing the web dashboard to lose connection.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a cloud upload success message.
2. **Alert Buzzer**: Add a buzzer (GPIO 15) and sound an alarm if the upload fails 3 times in a row.
3. **ThingSpeak Chart Integration**: Replace the local dashboard card with an embedded iframe chart from your real ThingSpeak channel.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| ThingSpeak returns error code -1 | DNS resolution failed | Verify that the ESP32 can resolve the domain `api.thingspeak.com` |
| Upload fails with return code 0 | Rate limit exceeded | Increase the `UPLOAD_INTERVAL_MS` to 20 or 30 seconds |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [67 - ESP32 Web Page DHT22 climate display](67-esp32-web-page-dht22-climate-display.md)
- [139 - ESP32 IoT Weather Station Node](139-esp32-iot-weather-station-node-dht22-bmp180-cloud-database-upload.md)
- [141 - ESP32 ThingSpeak Multi-Field Sensor database](141-esp32-thingspeak-multi-field-sensor-database-pressure-temp-gas.md) (Next project)
