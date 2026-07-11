# 141 - ThingSpeak Multi-Field Sensor database (Pressure + Temp + Gas)

Build an environmental and air quality database logger on the ESP32 that reads a DHT22 sensor, a BMP180 barometric pressure sensor on I2C (GPIO 21/22), and an MQ-2 gas sensor on analog GPIO 34, uploads all metrics simultaneously to a multi-field ThingSpeak channel, and hosts a diagnostic dashboard.

## Goal
Learn how to aggregate data from multiple heterogeneous sensors, structure multi-field HTTP query parameters, handle REST API submissions under rate-limiting constraints, and build local web dashboards.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It reads temperature and humidity from a DHT22, barometric pressure from a BMP180, and gas concentration from an MQ-2. Every 20 seconds, the ESP32 performs a multi-field update to ThingSpeak (Field 1: Temp, Field 2: Humidity, Field 3: Pressure, Field 4: Gas Level). A local webpage displays live values and sync details.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temperature & Humidity Sensor | `dht22` | Yes | Yes |
| BMP180 Barometric Pressure Sensor | `i2c_device` | No | Yes |
| MQ-2 Gas Sensor Module | `potentiometer` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use the simulated DHT22 widget and a potentiometer to simulate the analog MQ-2 gas concentration.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | OUT (Signal) | GPIO14 | Yellow | Temperature & humidity input |
| BMP180 Sensor | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C communication |
| MQ-2 Gas Sensor | AO (Analog Out) | GPIO34 | Green | Gas concentration input |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the MQ-2 sensor from the 5V Vin rail since it contains an internal heating element, but ensure its analog output is safe for the ESP32 3.3V pin.

## Code
```cpp
// ThingSpeak Multi-Field Sensor database (DHT22 + BMP180 + MQ-2 Air Station)
#include <WiFi.h>
#include <WebServer.h>
#include <HTTPClient.h>
#include <Wire.h>
#include <Adafruit_BMP085.h>
#include <DHT.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// ThingSpeak Configuration
const char* thingSpeakHost = "http://api.thingspeak.com/update";
const String thingSpeakWriteKey = "XYZ123456789"; // Replace with your Write API key

const int DHT_PIN = 14;
#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

Adafruit_BMP085 bmp;
const int MQ2_PIN = 34;

WebServer server(80);

// Global sensor values
float tempC = 0.0;
float humidPct = 0.0;
float pressureHpa = 1013.25;
int rawGas = 0;
float gasPct = 0.0;

unsigned long lastUploadTime = 0;
const unsigned long UPLOAD_INTERVAL_MS = 20000; // ThingSpeak free limit is 15 seconds
String cloudSyncStatus = "Idle";

// Read all environmental sensors
void readSensors() {
  tempC = dht.readTemperature();
  humidPct = dht.readHumidity();
  
  if (bmp.begin()) {
    pressureHpa = (float)bmp.readPressure() / 100.0;
  } else {
    // Simulated barometric drift if physical sensor is not present
    static float simPressure = 1012.5;
    simPressure += ((float)random(-3, 4) / 10.0);
    pressureHpa = simPressure;
  }
  
  rawGas = analogRead(MQ2_PIN);
  gasPct = ((float)rawGas / 4095.0) * 100.0; // Map 12-bit ADC to percentage
}

// HTTP GET multi-field update request
void uploadToThingSpeak() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    
    // Compile multi-field update query URL
    String url = String(thingSpeakHost) + "?api_key=" + thingSpeakWriteKey + 
                 "&field1=" + String(tempC, 1) + 
                 "&field2=" + String(humidPct, 1) + 
                 "&field3=" + String(pressureHpa, 1) + 
                 "&field4=" + String(gasPct, 1);
                 
    Serial.printf("[ThingSpeak] Request URL -> %s\n", url.c_str());
    http.begin(url);
    
    int httpResponseCode = http.GET();
    
    if (httpResponseCode > 0) {
      String response = http.getString();
      if (response == "0") {
        cloudSyncStatus = "Rate Limited (Wait 15s)";
        Serial.println("[ThingSpeak] Warning: Upload rate limited!");
      } else {
        cloudSyncStatus = "Success (Entry ID: " + response + ")";
        Serial.printf("[ThingSpeak] Upload successful. Entry ID: %s\n", response.c_str());
      }
    } else {
      cloudSyncStatus = "Failed (Error: " + String(httpResponseCode) + ")";
      Serial.printf("[ThingSpeak] Error sending request: %d\n", httpResponseCode);
    }
    http.end();
  } else {
    cloudSyncStatus = "WiFi Offline";
    Serial.println("[ThingSpeak] Sync blocked. WiFi offline.");
  }
}

// HTTP API endpoint returning JSON data
void handleGetWeather() {
  String json = "{\"temp\":" + String(tempC, 1) + 
                 ",\"humidity\":" + String(humidPct, 1) + 
                 ",\"pressure\":" + String(pressureHpa, 1) + 
                 ",\"gas\":" + String(gasPct, 1) + 
                 ",\"sync\":\"" + cloudSyncStatus + "\"}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Air Monitor Console</title>\n";
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
  html += "  .badge.warning { background-color: #92400e; color: #fef3c7; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Air Quality Station</h1>\n";
  html += "  <div id=\"syncBadge\" class=\"badge\">Idle</div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Temperature</div><div class=\"metric-val\" id=\"tempDisplay\">--.- &deg;C</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Humidity</div><div class=\"metric-val\" id=\"humidDisplay\">--.- %</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Pressure</div><div class=\"metric-val\" id=\"pressDisplay\">----.- hPa</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Gas Level</div><div class=\"metric-val\" id=\"gasDisplay\">--.- %</div></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">ThingSpeak Target: Fields 1-4 | Sync: 20 seconds</p>\n";
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
  html += "        document.getElementById('gasDisplay').innerText = data.gas.toFixed(1) + ' %';\n";
  
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
  html += "    setInterval(updateWeather, 2000);\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  dht.begin();
  
  if (!bmp.begin()) {
    Serial.println("[BMP Error] BMP180 sensor missing! Running in simulation mode.");
  }
  
  pinMode(MQ2_PIN, INPUT);
  
  Serial.println("\nESP32 Multi-Field Station starting...");
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
  
  // Refresh environmental variables
  readSensors();
  
  // Periodically upload data to the cloud (every 20 seconds)
  unsigned long now = millis();
  if (now - lastUploadTime >= UPLOAD_INTERVAL_MS) {
    lastUploadTime = now;
    uploadToThingSpeak();
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22 Sensor**, and **Potentiometer** (to simulate MQ-2) onto the canvas.
2. Wire DHT22 to **GPIO14** and Potentiometer to **GPIO34**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Slide the potentiometer to 50%. Verify that the Gas Level on the webpage updates to 50.0%.
6. Verify that the Serial Monitor outputs the complete request URL with `field1` to `field4` arguments.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[ThingSpeak] Request URL -> http://api.thingspeak.com/update?api_key=XYZ123456789&field1=24.0&field2=40.0&field3=1013.2&field4=50.0
[ThingSpeak] Upload successful. Entry ID: 45
```

## Expected Canvas Behavior
* Adjusting the simulated DHT22 and potentiometer widgets changes the values uploaded to the ThingSpeak cloud database in real time.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `readSensors()` | Collects readings from DHT22, BMP180, and MQ-2. |
| `rawGas / 4095.0 * 100.0` | Normalizes the 12-bit ADC value to a percentage representing concentration. |
| `field1=...&field2=...` | Packages multiple sensor values in a single HTTP GET request. |

## Hardware & Safety Concept: MQ-2 Heater Stabilization and Calibration
* **Sensor Heating**: MQ-2 gas sensors contain a chemical heating element that must warm up to operate correctly. When first powered, the sensor values will drift significantly. A **warm-up period** of at least 24 hours is recommended for hardware deployments before calibrating thresholds.
* **Analog Limits**: The MQ-2 sensor outputs a 0–5V analog signal. Connecting this directly to the ESP32's 3.3V pin can damage the board. Implement a resistive voltage divider (e.g., 10 kΩ and 20 kΩ) to scale the 5V signal down to 3.3V before feeding it into GPIO 34.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a safety warning indicator if the gas level exceeds 30%.
2. **Alert Buzzer**: Add a buzzer (GPIO 15) to sound an alarm when high gas levels are detected.
3. **ThingSpeak Multi-Chart Page**: Modify the webpage to embed four charts from ThingSpeak.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| ThingSpeak uploads fail with negative status codes | Wi-Fi disconnect | Check that your Wi-Fi is connected and you have internet access |
| Gas level stays at 100% | Wiring short | Verify that the MQ-2 analog output is connected to the correct GPIO pin (GPIO 34) |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [139 - ESP32 IoT Weather Station Node](139-esp32-iot-weather-station-node-dht22-bmp180-cloud-database-upload.md)
- [140 - ESP32 ThingSpeak Environment Monitor](140-esp32-thingspeak-environment-monitor-upload-temp-humidity-to-thingspeak-channel.md)
- [142 - Firebase Realtime Database Logger](142-firebase-realtime-database-logger-upload-current-state-to-firebase.md) (Next project)
