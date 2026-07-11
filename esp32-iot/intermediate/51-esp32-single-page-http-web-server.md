# 51 - Single Page HTTP Web Server

Build a standalone HTTP web server on the ESP32 that connects to a local WiFi network and hosts a responsive, styled single-page HTML dashboard displaying real-time system diagnostics (uptime, WiFi signal strength, and free heap memory).

## Goal
Learn how to host a local web server, compile inline CSS layouts, serve static HTML web payloads, and retrieve system diagnostics.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and runs a web server on port 80. Navigating to the ESP32's IP address (e.g. `http://10.10.0.3/`) in a web browser displays a dashboard detailing the device's signal strength (RSSI), uptime, and free heap memory.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// Single Page HTTP Web Server (Serving static HTML dashboard)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Host Web Server on standard HTTP Port 80
WebServer server(80);

// Root URL Route Handler ("/")
void handleRoot() {
  // 1. Gather ESP32 system diagnostics
  unsigned long uptimeSeconds = millis() / 1000;
  long rssi = WiFi.RSSI();
  uint32_t freeHeap = ESP.getFreeHeap();
  
  // 2. Compile responsive HTML page with modern styling
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>ESP32 IoT Node Dashboard</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 12px; padding: 30px; box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.5); max-width: 400px; width: 100%; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 24px; color: #38bdf8; margin-top: 0; text-align: center; border-bottom: 2px solid #334155; padding-bottom: 15px; }\n";
  html += "  .metric { display: flex; justify-content: space-between; margin: 15px 0; padding: 10px 0; border-bottom: 1px dashed #334155; }\n";
  html += "  .label { font-weight: bold; color: #94a3b8; }\n";
  html += "  .value { color: #f1f5f9; font-family: monospace; font-size: 16px; }\n";
  html += "  .footer { text-align: center; margin-top: 25px; font-size: 12px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>ESP32 System Status</h1>\n";
  html += "  <div class=\"metric\"><span class=\"label\">Uptime:</span><span class=\"value\">" + String(uptimeSeconds) + " s</span></div>\n";
  html += "  <div class=\"metric\"><span class=\"label\">WiFi Signal (RSSI):</span><span class=\"value\">" + String(rssi) + " dBm</span></div>\n";
  html += "  <div class=\"metric\"><span class=\"label\">Free Heap Memory:</span><span class=\"value\">" + String(freeHeap) + " bytes</span></div>\n";
  html += "  <p class=\"footer\">ESP32 Local Web Server | Port 80</p>\n";
  html += "</div>\n";
  
  html += "</body>\n</html>\n";
  
  // 3. Transmit the compiled HTML page to the client browser
  server.send(200, "text/html", html);
}

// Custom 404 page handler
void handleNotFound() {
  server.send(404, "text/plain", "Error 404: Route Not Found.");
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 Diagnostics Web Server Setup");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
  
  // Register URL routes
  server.on("/", handleRoot);
  server.onNotFound(handleNotFound);
  
  // Start the Web Server
  server.begin();
  
  Serial.print("Web Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  Serial.println("==================================\n");
}

void loop() {
  // Handle incoming HTTP client requests
  server.handleClient();
  
  delay(2); // Yield to system
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Note the printed IP address (e.g. `http://10.10.0.3`).
4. Open your web browser and navigate to the IP. Verify that the system diagnostics page loads.

## Expected Output
Serial Monitor:
```
==================================
ESP32 Diagnostics Web Server Setup
==================================
WiFi Connected successfully.
Web Server active. Connect at: http://10.10.0.3
==================================
```

Web Browser screen shows:
* A dark-themed card titled "ESP32 System Status" containing the live values for Uptime, WiFi Signal (RSSI), and Free Heap Memory.

## Expected Canvas Behavior
* Navigating to the page updates the diagnostics values logged in the web console.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WebServer server(80)` | Instantiates a web server object listening on standard HTTP port 80. |
| `server.on("/", handleRoot)` | Binds the root URL route to the handler function. |
| `ESP.getFreeHeap()` | Queries the ESP32 system allocator for the current available RAM in bytes. |
| `server.send(200, "text/html", html)` | Transmits the HTTP response (code 200, content type, and HTML body). |

## Hardware & Safety Concept: Free Heap Memory Monitoring in Servers
Every time a web server handles a connection, it allocates memory to compile and transmit pages. If memory is not managed correctly (e.g. leaking socket allocations or creating excessively large string copies), the heap memory will fragment and exhaust the RAM. Monitoring the **Free Heap** allows identifying potential memory leaks before the ESP32 crashes.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the count of web requests.
2. **Auto-refresh page**: Add a meta tag `<meta http-equiv="refresh" content="5">` to the HTML header to automatically refresh the page every 5 seconds.
3. **RSSI Signal warning**: Sound a warning beep on a buzzer (GPIO 15) if the connection drops.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The webpage fails to load | Client on different network | Ensure your computer and the ESP32 are connected to the same WiFi network |
| Page takes long to load | Large string copies | Use smaller HTML structures or compile layouts using static character arrays |
| Server is slow to respond | Blocking calls in loop | Remove any blocking `delay()` calls inside the main loop |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [44 - ESP32 Web-based LED toggle](../beginner/44-esp32-web-based-led-toggle.md)
- [52 - ESP32 Web Page LED Toggle button](52-esp32-web-page-led-toggle-button.md) (Next project)
- [62 - ESP32 Web Server serving files from SPIFFS](62-esp32-web-server-serving-files-from-spiffs.md)
