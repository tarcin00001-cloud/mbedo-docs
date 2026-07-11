# 67 - Web Page DHT22 Climate Display (auto-update)

Build a web-connected climate monitor on the ESP32 that hosts a control page, reads temperature and humidity from a DHT22 sensor on GPIO 4, serves a JSON API endpoint, and uses client-side JavaScript (AJAX polling) to refresh the display parameters every 2 seconds.

## Goal
Learn how to sample DHT22 sensors, format JSON climate packets, implement client-side AJAX polling loops, and display values on responsive cards.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a web server on port 80. A DHT22 sensor is on GPIO 4. Navigating to the ESP32's IP address displays a webpage showing the live temperature in degrees Celsius and humidity percentage. JavaScript on the page polls `/api/climate` every 2 seconds, updating the displayed values.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | DATA | GPIO4 | Yellow | Temp & Humidity input |
| Shared power | VCC / GND | 3V3 / GND | Red / Black | Sensor power rails |

> **Wiring tip:** Connect the DHT22 data pin directly to GPIO 4.

## Code
```cpp
// Web Page DHT22 climate display (Asynchronous climate dashboard)
#include <WiFi.h>
#include <WebServer.h>
#include <DHT.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int DHT_PIN = 4;
#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

WebServer server(80);

// 1. Root URL Handler ("/")
// Serves the HTML dashboard containing CSS and the AJAX polling loop
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Climate Dashboard</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .container { max-width: 600px; width: 100%; }\n";
  html += "  h1 { font-size: 26px; color: #38bdf8; text-align: center; margin-bottom: 30px; }\n";
  html += "  .grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(240px, 1fr)); gap: 20px; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 12px; padding: 25px; text-align: center; border: 1px solid #334155; box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.3); }\n";
  html += "  .card-temp h2 { color: #f43f5e; } .card-hum h2 { color: #38bdf8; }\n";
  html += "  .value { font-size: 44px; font-weight: 800; margin: 15px 0; font-family: monospace; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"container\">\n";
  html += "  <h1>ESP32 Climate Control</h1>\n";
  html += "  <div class=\"grid\">\n";
  
  // Temperature Card
  html += "    <div class=\"card card-temp\">\n";
  html += "      <h2>Temperature</h2>\n";
  html += "      <div class=\"value\" id=\"tempVal\">--.- &deg;C</div>\n";
  html += "    </div>\n";
  
  // Humidity Card
  html += "    <div class=\"card card-hum\">\n";
  html += "      <h2>Humidity</h2>\n";
  html += "      <div class=\"value\" id=\"humVal\">--.- %</div>\n";
  html += "    </div>\n";
  
  html += "  </div>\n";
  html += "  <p class=\"footer\">Asynchronous updates polling every 2 seconds | Port 80</p>\n";
  html += "</div>\n";
  
  // 2. Client-side AJAX polling script using Fetch API
  html += "<script>\n";
  html += "  function updateClimate() {\n";
  html += "    fetch('/api/climate')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('tempVal').innerHTML = data.temp.toFixed(1) + ' &deg;C';\n";
  html += "        document.getElementById('humVal').innerText = data.hum.toFixed(1) + ' %';\n";
  html += "      })\n";
  html += "      .catch(err => console.error('Error fetching climate data:', err));\n";
  html += "  }\n";
  
  html += "  // Start polling loop once page finishes loading\n";
  html += "  window.onload = function() {\n";
  html += "    updateClimate();\n";
  html += "    setInterval(updateClimate, 2000); // Poll once every 2 seconds\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// 3. API Telemetry URL Handler ("/api/climate")
void handleClimateAPI() {
  float temp = dht.readTemperature();
  float hum = dht.readHumidity();
  
  // Check if readings are valid
  if (isnan(temp) || isnan(hum)) {
    temp = 0.0;
    hum = 0.0;
  }
  
  // Compile JSON telemetry string
  String json = "{\"temp\":" + String(temp, 1) + 
                 ",\"hum\":" + String(hum, 1) + "}";
  
  server.send(200, "application/json", json);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  dht.begin();
  
  Serial.println("\nESP32 Climate Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Register routes
  server.on("/", handleRoot);
  server.on("/api/climate", handleClimateAPI);
  
  server.begin();
  Serial.print("Web Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **DHT22** onto the canvas.
2. Wire DHT22 to **GPIO4**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Adjust the DHT22 temperature and humidity sliders. Watch the values update automatically on the webpage.

## Expected Output
Serial Monitor:
```
ESP32 Climate Server Starting...
....
WiFi Connected.
Web Server active. Connect at: http://10.10.0.3
```

Browser Console (Inspect Network tab):
* Continuous lightweight GET requests to `/api/climate` returning JSON strings (`{"temp":24.5,"hum":45.0}`).

## Expected Canvas Behavior
* Adjusting the DHT22 sliders updates the temperature and humidity values on the webpage in real time.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `dht.readTemperature()` | Reads the temperature value in degrees Celsius from the DHT22 sensor. |
| `setInterval(updateClimate, 2000)` | Registers a browser timer to call the update function every 2000 ms. |
| `data.temp` | Resolves the temperature key parameter from the parsed JSON response object. |
| `server.send(200, "application/json", ...)` | Serves the climate payload with the correct JSON MIME type. |

## Hardware & Safety Concept: Sensor Read Delays and Network Load
DHT22 sensors use a single-wire digital protocol that requires about 2 seconds between consecutive readings to obtain stable data. Querying the sensor faster will return stale data or error values. By configuring the client-side AJAX polling interval to 2 seconds, the server matches the hardware limits, avoiding CPU overhead.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the current climate readings.
2. **Climate alert buzzer**: Add a buzzer (GPIO 15) and sound an alarm if the temperature exceeds 35.0 °C.
3. **Fahrenheit Toggle**: Add a checkbox on the webpage to toggle the temperature display between Celsius and Fahrenheit.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |
| Values display as 0.0 or nan | DHT22 read failure | Confirm that the sensor data pin is connected to GPIO 4 |
| Web page responds slowly | Blocking delays | Avoid using blocking `delay()` calls inside the main loop |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [49 - ESP32 Wireless DHT22 logger](../beginner/49-esp32-wireless-dht22-logger.md)
- [61 - ESP32 Web Server AJAX Telemetry Auto-refresh](61-esp32-web-server-ajax-telemetry-auto-refresh.md)
- [68 - ESP32 Web Page MQ-2 Gas sensor gauge display](68-esp32-web-page-mq-2-gas-sensor-gauge-display.md) (Next project)
