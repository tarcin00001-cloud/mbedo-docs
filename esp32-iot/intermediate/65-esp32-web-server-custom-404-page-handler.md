# 65 - Web Server Custom 404 page handler

Configure a web server on the ESP32 that registers a custom `onNotFound` route handler, intercepts request errors for unregistered URL paths, and serves a responsive, dark-themed HTML 404 page displaying the requested path and a link to return to the root dashboard.

## Goal
Learn how to intercept HTTP routing errors, configure response status codes (HTTP 404 Not Found), query request URI strings, and design fallbacks.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a web server on port 80. Navigating to the root `/` displays a welcome dashboard. Navigating to any unregistered path (e.g. `/admin` or `/temp`) triggers the custom 404 page, which displays the requested path and a link to return to the home page.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// Web Server Custom 404 page handler (Route Interceptor)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

WebServer server(80);

// Root URL Route Handler ("/")
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>ESP32 Home</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 12px; padding: 30px; box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.4); max-width: 400px; width: 100%; text-align: center; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 24px; color: #38bdf8; margin-top: 0; }\n";
  html += "  p { color: #94a3b8; line-height: 1.6; }\n";
  html += "  .btn { display: inline-block; padding: 12px 24px; font-size: 14px; font-weight: bold; color: white; background-color: #38bdf8; border: none; border-radius: 6px; cursor: pointer; text-decoration: none; transition: background-color 0.2s; }\n";
  html += "  .btn:hover { background-color: #0ea5e9; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>ESP32 IoT Node</h1>\n";
  html += "  <p>Welcome to the main dashboard. The server is active and responding to valid routes.</p>\n";
  html += "  <p>To test the custom error page, type an invalid path in the URL bar (e.g. /random-page).</p>\n";
  html += "  <a href=\"/random-page\" class=\"btn\">Go to Invalid Path</a>\n";
  html += "</div>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// Custom 404 URL Route Handler (onNotFound)
void handleNotFound() {
  // 1. Log the failed routing request
  String requestedPath = server.uri();
  Serial.print("[Server Alert] Client requested invalid route: ");
  Serial.println(requestedPath);
  
  // 2. Compile custom dark-themed error page
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>404 - Page Not Found</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.5); max-width: 400px; width: 100%; text-align: center; border: 1px solid #dc2626; }\n";
  html += "  .error-code { font-size: 72px; font-weight: 800; color: #dc2626; margin: 0; line-height: 1; }\n";
  html += "  h1 { font-size: 22px; color: #f1f5f9; margin-top: 15px; margin-bottom: 10px; }\n";
  html += "  .path { display: inline-block; background-color: #334155; padding: 4px 10px; border-radius: 4px; font-family: monospace; color: #f43f5e; font-size: 14px; margin-bottom: 20px; }\n";
  html += "  p { color: #94a3b8; margin-bottom: 30px; line-height: 1.6; }\n";
  html += "  .btn { display: inline-block; padding: 12px 24px; font-size: 14px; font-weight: bold; color: white; background-color: #38bdf8; border: none; border-radius: 6px; cursor: pointer; text-decoration: none; transition: background-color 0.2s; }\n";
  html += "  .btn:hover { background-color: #0ea5e9; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <div class=\"error-code\">404</div>\n";
  html += "  <h1>Page Not Found</h1>\n";
  html += "  <span class=\"path\">" + requestedPath + "</span>\n";
  html += "  <p>The requested path could not be located on this ESP32 web server. Please confirm the address or click the button below to return to the home page.</p>\n";
  html += "  <a href=\"/\" class=\"btn\">Back to Dashboard</a>\n";
  html += "</div>\n";
  
  html += "</body>\n</html>\n";
  
  // 3. Transmit the 404 payload
  // Ensure the status code is set to 404 (Not Found)
  server.send(404, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\nESP32 Custom 404 Web Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Register routes
  server.on("/", handleRoot);
  
  // 4. Register custom 404 error handler
  server.onNotFound(handleNotFound);
  
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
3. Open your web browser and navigate to the printed IP address. Verify that the welcome page loads.
4. Click the "Go to Invalid Path" button. Verify that the custom 404 page displays.

## Expected Output
Serial Monitor:
```
ESP32 Custom 404 Web Server Starting...
....
WiFi Connected.
Web Server active. Connect at: http://10.10.0.3

[Server Alert] Client requested invalid route: /random-page
```

Browser Page Layout (On invalid route):
* A dark-themed card container with a red border displaying a large red `404` and the text `Page Not Found` with a link pointing back to the home page.

## Expected Canvas Behavior
* Navigating to an invalid route logs the requested route on the Serial Monitor.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `server.onNotFound(...)` | Tells the web server to execute the specified handler if a requested route is not registered. |
| `server.uri()` | Queries the current requested URL path (e.g. `/random-page`). |
| `server.send(404, ...)` | Transmits the response with the HTTP 404 (Not Found) status code. |

## Hardware & Safety Concept: HTTP Error Codes and Web Design
Web browsers and search crawlers evaluate status codes to determine response outcomes. If a server displays a "Not Found" message but returns a `200 OK` status code, crawlers will index the error page, polluting search indexes. Returning a **404 Not Found** status code ensures search indexes remain clean.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a count of failed route requests.
2. **Auto-redirect**: Add a JavaScript timer to automatically redirect the client browser back to `/` after 5 seconds of displaying the 404 error page.
3. **Buzzer warning tone**: Sound a warning beep on a buzzer (GPIO 15) when a 404 error is generated.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| standard browser error page shows | status code wrong | Confirm that `server.send(404, ...)` is called with the correct HTML body |
| Custom 404 page does not load | route registered | Check if the target path matches a registered route (registered routes do not trigger the 404 handler) |
| Web page responds slowly | Blocking delays | Avoid using blocking `delay()` calls inside the main loop |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [51 - ESP32 Single Page HTTP Web Server](51-esp32-single-page-http-web-server.md)
- [62 - ESP32 Web Server serving files from SPIFFS](62-esp32-web-server-serving-files-from-spiffs.md)
- [64 - ESP32 Web Server HTTP basic authentication lock](64-esp32-web-server-http-basic-authentication-lock.md)
