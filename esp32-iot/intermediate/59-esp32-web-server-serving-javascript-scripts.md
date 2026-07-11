# 59 - Web Server Serving JavaScript Scripts

Build a web server on the ESP32 that separates presentation logic from client-side scripts by serving a JavaScript file via a dedicated `/script.js` route, referencing it in the root HTML file, and using it to request uptime data from `/api/uptime`.

## Goal
Learn how to separate HTML and JavaScript files, configure Content-Type headers for scripts (`application/javascript`), query API endpoints, and update elements on a page.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a web server on port 80. Navigating to `/` loads an HTML page that references `/script.js`. The JavaScript runs in the browser, requesting the system uptime from `/api/uptime` every second and updating a text label on the page without reloading the webpage.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// Web Server serving JavaScript scripts (Separated Javascript asset serving)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

WebServer server(80);

// 1. Separate JavaScript file payload
// Contains the script that queries /api/uptime and updates the text label
const char JS_CONTENT[] PROGMEM = 
"function fetchUptime() {"
"  fetch('/api/uptime')" // Send HTTP GET request to API route
"    .then(response => response.text())" // Read response as text
"    .then(data => {"
"      document.getElementById('uptimeVal').innerText = data + ' seconds';" // Update DOM element
"    })"
"    .catch(err => console.error('Fetch error:', err));"
"}"
"// Fetch uptime immediately and then repeat every 1 second\n"
"window.onload = function() {"
"  fetchUptime();"
"  setInterval(fetchUptime, 1000);"
"};";

// 2. Root URL Handler ("/")
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>ESP32 JavaScript Server</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 12px; padding: 30px; box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.4); max-width: 350px; width: 100%; text-align: center; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-bottom: 20px; }\n";
  html += "  .uptime-display { font-size: 20px; font-weight: bold; color: #10b981; margin: 25px 0; font-family: monospace; }\n";
  html += "</style>\n";
  
  // Link to the Javascript script file
  html += "<script src=\"/script.js\"></script>\n";
  
  html += "</head>\n<body>\n";
  html += "  <div class=\"card\">\n";
  html += "    <h1>JavaScript Stream</h1>\n";
  html += "    <p>Uptime data is fetched from the ESP32 in the background using JavaScript:</p>\n";
  html += "    <div class=\"uptime-display\" id=\"uptimeVal\">Loading...</div>\n";
  html += "    <p style=\"font-size: 12px; color: #64748b;\">The page does not reload during updates.</p>\n";
  html += "  </div>\n";
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// 3. JavaScript URL handler ("/script.js")
void handleJS() {
  server.sendHeader("Cache-Control", "max-age=86400");
  server.send(200, "application/javascript", JS_CONTENT);
}

// 4. API Uptime URL handler ("/api/uptime")
void handleUptimeAPI() {
  String uptimeSec = String(millis() / 1000);
  
  // Send simple text/plain response containing the uptime value
  server.send(200, "text/plain", uptimeSec);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\nESP32 JavaScript Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Register routes
  server.on("/", handleRoot);
  server.on("/script.js", handleJS);
  server.on("/api/uptime", handleUptimeAPI);
  
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
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Open your web browser and navigate to the printed IP address. Verify that the uptime counter updates dynamically on the screen.
4. Open the browser's developer console to watch the script execute background requests.

## Expected Output
Serial Monitor:
```
ESP32 JavaScript Server Starting...
....
WiFi Connected.
Web Server active. Connect at: http://10.10.0.3
```

Browser Page Layout:
```
JavaScript Stream
Uptime data is fetched from the ESP32 in the background using JavaScript:
12 seconds  (counter increments every second)
```

## Expected Canvas Behavior
* Navigating to the page triggers the script to request updates in the background, logging the requests.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `<script src="/script.js">` | Instructs the browser to fetch and execute the JavaScript file from the server. |
| `server.send(200, "application/javascript", ...)` | Serves the JavaScript file with the correct MIME type so the browser can run it. |
| `fetch('/api/uptime')` | Standard JavaScript API to send HTTP GET requests to the specified endpoint. |
| `setInterval(fetchUptime, 1000)` | Registers a timer in the browser to run the fetch function every second. |

## Hardware & Safety Concept: Separating Assets and Cache Optimizations
Serving HTML page layouts that contain inline CSS and JavaScript files increases the size of each page request, consuming more memory on the ESP32. Separating CSS and JavaScript files into separate files allows:
1. **Asset Separation**: Keeps code organized and reduces page compile overhead.
2. **Cache Optimizations**: The browser caches static CSS and JavaScript files, requesting only dynamic raw data (like `/api/uptime`), reducing network load.

## Try This! (Challenges)
1. **OLED Request display**: Add an OLED screen (Project 60) and display the count of background API requests.
2. **Signal strength API**: Add an API route `/api/rssi` and display the signal strength on the webpage.
3. **SPIFFS integration**: Store the JavaScript file in the flash memory file system (SPIFFS) and serve it from there (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The counter displays "Loading..." permanently | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript console errors |
| API requests return 404 | Route not registered | Verify that the `/api/uptime` route is registered in setup |
| The ESP32 crashes on loop | String allocation leak | Ensure JavaScript strings are stored in flash memory using `PROGMEM` |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [58 - ESP32 Web Server serving CSS styles](58-esp32-web-server-serving-css-styles.md)
- [60 - ESP32 Web Server AJAX LED toggle (no page refresh)](60-esp32-web-server-ajax-led-toggle-no-page-refresh.md) (Next project)
- [62 - ESP32 Web Server serving files from SPIFFS](62-esp32-web-server-serving-files-from-spiffs.md)
