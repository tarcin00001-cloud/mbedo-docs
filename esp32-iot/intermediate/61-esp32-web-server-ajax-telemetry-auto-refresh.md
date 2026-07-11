# 61 - Web Server AJAX Telemetry Auto-refresh

Build an auto-updating telemetry dashboard on the ESP32 that connects to a local WiFi network, reads an analog potentiometer sensor, serves a JSON API endpoint, and uses client-side JavaScript (AJAX polling) to refresh the display parameters every second.

## Goal
Learn how to implement client-side AJAX polling loops, format sensor values as JSON telemetry packets, update DOM elements, and build remote gauges.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a web server on port 80. A potentiometer is on GPIO 34. Navigating to the ESP32's IP address displays a dashboard showing the raw ADC reading and calculated voltage. JavaScript on the page polls `/api/sensor` every second, updating the displayed values and a progress bar immediately without reloading the page.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Wiper | GPIO34 | Yellow | Analog sensor signal |
| Shared power | VCC / GND | 3V3 / GND | Red / Black | Sensor power rails |

> **Wiring tip:** Connect the potentiometer wiper to GPIO 34.

## Code
```cpp
// Web Server AJAX Telemetry Auto-refresh (Asynchronous sensor dashboard)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int POT_PIN = 34;

WebServer server(80);

// 1. Root URL Handler ("/")
// Serves the HTML dashboard containing CSS and the AJAX polling loop
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>ESP32 Telemetry Monitor</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 400px; width: 100%; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; margin-bottom: 25px; text-align: center; }\n";
  
  html += "  .metric { display: flex; justify-content: space-between; margin: 15px 0; padding: 10px 0; border-bottom: 1px dashed #334155; }\n";
  html += "  .label { font-weight: bold; color: #94a3b8; }\n";
  html += "  .value { color: #f1f5f9; font-family: monospace; font-size: 18px; }\n";
  
  // Progress bar to visualize value
  html += "  .progress-bg { background-color: #475569; height: 16px; border-radius: 8px; overflow: hidden; margin-top: 20px; }\n";
  html += "  .progress-bar { background-color: #10b981; height: 100%; width: 0%; transition: width 0.3s ease; }\n";
  html += "  .footer { text-align: center; margin-top: 25px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Live Sensor Monitor</h1>\n";
  html += "  <div class=\"metric\"><span class=\"label\">Raw ADC (0-4095):</span><span class=\"value\" id=\"rawVal\">0</span></div>\n";
  html += "  <div class=\"metric\"><span class=\"label\">Voltage (0-3.3V):</span><span class=\"value\" id=\"voltVal\">0.00 V</span></div>\n";
  
  // Visual bar
  html += "  <div class=\"progress-bg\">\n";
  html += "    <div id=\"progBar\" class=\"progress-bar\"></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Data updates automatically every second without reloading.</p>\n";
  html += "</div>\n";
  
  // 2. Client-side AJAX polling script using Fetch API
  html += "<script>\n";
  html += "  function updateTelemetry() {\n";
  html += "    fetch('/api/sensor')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('rawVal').innerText = data.raw;\n";
  html += "        document.getElementById('voltVal').innerText = data.voltage + ' V';\n";
  
  // Update progress bar width dynamically
  html += "        const percent = (data.raw / 4095) * 100;\n";
  html += "        document.getElementById('progBar').style.width = percent + '%';\n";
  html += "      })\n";
  html += "      .catch(err => console.error('Error fetching telemetry:', err));\n";
  html += "  }\n";
  
  html += "  // Start polling loop once page finishes loading\n";
  html += "  window.onload = function() {\n";
  html += "    updateTelemetry();\n";
  html += "    setInterval(updateTelemetry, 1000); // Poll once every 1000 ms\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// 3. API Telemetry URL Handler ("/api/sensor")
void handleSensorAPI() {
  int potRaw = analogRead(POT_PIN);
  float voltage = potRaw * (3.3 / 4095.0);
  
  // Compile JSON telemetry string
  String json = "{\"raw\":" + String(potRaw) + 
                 ",\"voltage\":" + String(voltage, 2) + "}";
  
  server.send(200, "application/json", json);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(POT_PIN, INPUT);
  
  Serial.println("\nESP32 Telemetry Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Register routes
  server.on("/", handleRoot);
  server.on("/api/sensor", handleSensorAPI);
  
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
1. Drag **ESP32 DevKitC** and **Potentiometer** onto the canvas.
2. Wire Potentiometer to **GPIO34**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Slide the potentiometer widget. Watch the values and the progress bar update automatically in real time.

## Expected Output
Serial Monitor:
```
ESP32 Telemetry Server Starting...
....
WiFi Connected.
Web Server active. Connect at: http://10.10.0.3
```

Browser Console (Inspect Network tab):
* Continuous lightweight GET requests to `/api/sensor` returning JSON strings (`{"raw":2048,"voltage":1.65}`).

## Expected Canvas Behavior
* Adjusting the potentiometer updates the progress bar and values on the webpage in real time.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `setInterval(updateTelemetry, 1000)` | Registers a browser timer to call the update function every 1000 ms. |
| `data.raw` | Resolves the raw key parameter from the parsed JSON response object. |
| `style.width = percent + '%'` | Modifies the CSS width property of the progress bar to show the value. |
| `server.send(200, "application/json", ...)` | Serves the sensor payload with the correct JSON MIME type. |

## Hardware & Safety Concept: Asynchronous Polling Rates and Network Load
Polling updates telemetry values without user interaction. When configuring polling rates:
1. **Poll Intervals**: Set the interval to a reasonable frequency (e.g. 1 Hz). Polling too fast (e.g. every 50 ms) will flood the ESP32 server with HTTP requests, exhausting sockets and causing connections to drop.
2. **WebSockets fallback**: For higher update rates (e.g. 10 Hz to 50 Hz), use WebSockets (Project 71) instead of HTTP polling.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the current sensor values.
2. **Overheat buzzer alert**: Add a buzzer (GPIO 15) and sound an alarm if the voltage exceeds 3.0V.
3. **Multi-sensor API**: Connect a DHT22 sensor (Project 49) and add temperature and humidity fields to the JSON telemetry response.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |
| API requests return 404 | Route not registered | Verify that the `/api/sensor` route is registered in setup |
| Progress bar does not move | Division error | Check the javascript math, ensuring raw is not divided by zero |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [47 - ESP32 Wireless Potentiometer display](../beginner/47-esp32-wireless-potentiometer-display.md)
- [60 - ESP32 Web Server AJAX LED toggle (no page refresh)](60-esp32-web-server-ajax-led-toggle-no-page-refresh.md)
- [73 - ESP32 WebSocket Sensor Streamer](73-esp32-websocket-sensor-streamer.md)
