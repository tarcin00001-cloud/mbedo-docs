# 144 - Adafruit IO Dashboard logger (Push values to Adafruit IO dashboard)

Build an environmental telemetry logger on the ESP32 that samples a DHT22 climate sensor on GPIO 14 and an LDR light sensor on analog GPIO 34, uploads both values to separate Adafruit IO dashboard feeds using secure HTTP REST POST requests, and hosts a local diagnostic status page.

## Goal
Learn how to interact with the Adafruit IO REST API, configure custom HTTP header authentication (`X-AIO-Key`), serialize feed data payloads, and manage multi-feed schedule intervals.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A DHT22 sensor is on GPIO 14, and an LDR on GPIO 34. Every 15 seconds, the ESP32 reads temperature and light levels. It sends two separate HTTP POST requests containing these values to Adafruit IO feeds (`https://io.adafruit.com/api/v2/[username]/feeds/[feed]/data`). A local webpage displays current values and sync logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temperature & Humidity Sensor | `dht22` | Yes | Yes |
| Photoresistor (LDR) | `photoresistor` | Yes | Yes |
| 10 kΩ Resistor (voltage divider) | `resistor` | No | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | OUT (Signal) | GPIO14 | Yellow | Climate sensor signal |
| LDR Sensor | Pin 1 / Pin 2 | GPIO34 / 3V3 | Green / Red | Light level signal |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 10 kΩ pull-down resistor in series with the LDR to form a voltage divider circuit.

## Code
```cpp
// Adafruit IO Dashboard logger (DHT22 + LDR + Adafruit IO REST Client)
#include <WiFi.h>
#include <WebServer.h>
#include <HTTPClient.h>
#include <DHT.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Adafruit IO credentials
const String aioUsername = "your_username";
const String aioKey = "aio_XYZ123456789"; // Place your Adafruit IO key here

const int DHT_PIN = 14;
#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

const int LDR_PIN = 34;

WebServer server(80);

// Global values
float currentTemp = 0.0;
int currentLight = 0;

unsigned long lastUploadTime = 0;
const unsigned long UPLOAD_INTERVAL_MS = 15000; // Log data every 15 seconds
String adafruitSyncStatus = "Idle";

// Send HTTP POST request to push data to an Adafruit IO feed
bool pushToAdafruitIO(String feedKey, float value) {
  if (WiFi.status() != WL_CONNECTED) return false;
  
  HTTPClient http;
  
  // Endpoint format: https://io.adafruit.com/api/v2/[username]/feeds/[feed_key]/data
  String url = "https://io.adafruit.com/api/v2/" + aioUsername + "/feeds/" + feedKey + "/data";
  
  http.begin(url);
  http.addHeader("Content-Type", "application/json");
  http.addHeader("X-AIO-Key", aioKey); // Authenticate using X-AIO-Key header
  
  String payload = "{\"value\":" + String(value, 1) + "}";
  
  int responseCode = http.POST(payload);
  http.end();
  
  return (responseCode == 200 || responseCode == 201);
}

// HTTP API endpoint returning JSON data
void handleGetTelemetry() {
  String json = "{\"temp\":" + String(currentTemp, 1) + 
                 ",\"light\":" + String(currentLight) + 
                 ",\"sync\":\"" + adafruitSyncStatus + "\"}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Adafruit IO Logger</title>\n";
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
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Adafruit IO Monitor</h1>\n";
  html += "  <div id=\"syncBadge\" class=\"badge\">Idle</div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Temperature</div><div class=\"metric-val\" id=\"tempDisplay\">--.- &deg;C</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Light Level</div><div class=\"metric-val\" id=\"lightDisplay\">-- %</div></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Adafruit IO Feeds: temperature, light | Sync: 15s</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateTelemetry() {\n";
  html += "    fetch('/api/telemetry')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('tempDisplay').innerHTML = data.temp.toFixed(1) + ' &deg;C';\n";
  html += "        document.getElementById('lightDisplay').innerText = data.light + ' %';\n";
  
  html += "        const bdgEl = document.getElementById('syncBadge');\n";
  html += "        bdgEl.innerText = data.sync;\n";
  html += "        if (data.sync.startsWith('Success')) {\n";
  html += "          bdgEl.className = 'badge synced';\n";
  html += "        } else {\n";
  html += "          bdgEl.className = 'badge';\n";
  html += "        }\n";
  // Handled network diagnostics safely
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateTelemetry();\n";
  html += "    setInterval(updateTelemetry, 2000);\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  dht.begin();
  pinMode(LDR_PIN, INPUT);
  
  Serial.println("\nESP32 Adafruit IO client starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/telemetry", handleGetTelemetry);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // Refresh environmental variables
  currentTemp = dht.readTemperature();
  int rawLight = analogRead(LDR_PIN);
  currentLight = map(rawLight, 0, 4095, 0, 100);
  
  // Periodically upload data to Adafruit IO feeds (every 15 seconds)
  unsigned long now = millis();
  if (now - lastUploadTime >= UPLOAD_INTERVAL_MS) {
    lastUploadTime = now;
    
    if (!isnan(currentTemp)) {
      Serial.println("[Adafruit IO] Synchronizing feeds...");
      bool tOk = pushToAdafruitIO("temperature", currentTemp);
      bool lOk = pushToAdafruitIO("light", currentLight);
      
      if (tOk && lOk) {
        adafruitSyncStatus = "Success";
        Serial.println("[Adafruit IO] Both feeds updated successfully.");
      } else {
        adafruitSyncStatus = "Upload Error";
        Serial.println("[Adafruit IO] Failed to update one or more feeds.");
      }
    }
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22 Sensor**, and **Potentiometer** (to simulate the LDR) onto the canvas.
2. Wire DHT22 to **GPIO14** and Potentiometer to **GPIO34**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Slide the potentiometer to 70%. Verify that the Light Level on the webpage updates to 70%.
6. Verify that the Serial Monitor outputs `Synchronizing feeds` and indicates upload status.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Adafruit IO] Synchronizing feeds...
[Adafruit IO] Both feeds updated successfully.
```

## Expected Canvas Behavior
* Adjusting the simulated DHT22 and potentiometer sliders dynamically updates the telemetry values sent to the Adafruit IO cloud API.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pushToAdafruitIO(...)` | Initiates the HTTP POST request to Adafruit IO. |
| `http.addHeader("X-AIO-Key", aioKey)` | Adds the Adafruit IO authentication header key. |
| `responseCode == 201` | Returns true if the data point was successfully created on the server. |

## Hardware & Safety Concept: REST API Headers and Platform Rate Limits
* **Header Authentication**: Unlike public endpoints that allow anonymous data submissions, private dashboards require authentication. Adafruit IO uses a custom header key (`X-AIO-Key`) to verify permissions, protecting your dashboard feeds from unauthorized updates.
* **Rate Limits**: Adafruit IO free accounts are limited to a maximum of 30 data points per minute (average of 1 point every 2 seconds). Sending too many rapid requests will trigger a HTTP 429 rate limit block. Structure your code to combine updates or use a longer polling interval (e.g. 15 seconds) to stay well within limits.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the sync status.
2. **Alert sound**: Add a buzzer (GPIO 15) to sound a chime on upload errors.
3. **Adafruit IO MQTT client**: Rewrite the code to use MQTT instead of HTTP REST to publish data, which reduces WiFi overhead.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Feeds fail to upload | Invalid username or API key | Double check your Adafruit IO username and AIO Key in the code |
| HTTP response returns 429 | Rate limit exceeded | Increase the `UPLOAD_INTERVAL_MS` to 20 seconds to prevent blocking |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [140 - ESP32 ThingSpeak Environment Monitor](140-esp32-thingspeak-environment-monitor-upload-temp-humidity-to-thingspeak-channel.md)
- [143 - Firebase Authentication Access Controller](143-firebase-authentication-access-controller.md)
- [145 - Adafruit IO Remote Actuator controller](145-esp32-adafruit-io-remote-actuator-controller.md) (Next project)
