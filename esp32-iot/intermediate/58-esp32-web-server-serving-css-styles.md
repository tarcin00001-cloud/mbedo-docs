# 58 - Web Server Serving CSS Styles

Build a web server on the ESP32 that separates HTML structure from CSS presentation by serving a custom stylesheet via a dedicated `/style.css` route and referencing it in the root dashboard layout.

## Goal
Learn how to separate HTML and CSS payloads, configure Content-Type headers for stylesheets (`text/css`), and cache static assets.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a web server on port 80. Navigating to the root route `/` loads a styled dashboard. The browser automatically requests the stylesheet from `/style.css` to render the page, verifying the separation of concerns.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// Web Server serving CSS styles (Separated CSS asset serving)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

WebServer server(80);

// 1. Separate CSS stylesheet payload
// Served as a static string via /style.css
const char CSS_CONTENT[] PROGMEM = 
"body { "
"  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; "
"  background-color: #0f172a; "
"  color: #f8fafc; "
"  margin: 0; "
"  padding: 20px; "
"  display: flex; "
"  justify-content: center; "
"  align-items: center; "
"  min-height: 100vh; "
"}"
".card { "
"  background-color: #1e293b; "
"  border-radius: 16px; "
"  padding: 40px; "
"  box-shadow: 0 10px 30px -10px rgba(0,0,0,0.5); "
"  max-width: 450px; "
"  width: 100%; "
"  border: 1px solid #334155; "
"  text-align: center; "
"}"
"h1 { "
"  color: #38bdf8; "
"  font-size: 26px; "
"  margin-bottom: 20px; "
"}"
"p { "
"  color: #94a3b8; "
"  line-height: 1.6; "
"  font-size: 16px; "
"}"
".badge { "
"  display: inline-block; "
"  background-color: #0369a1; "
"  color: #e0f2fe; "
"  padding: 6px 14px; "
"  border-radius: 20px; "
"  font-size: 14px; "
"  font-weight: 600; "
"  margin-top: 15px; "
"}";

// 2. HTML page referencing the stylesheet
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>ESP32 Styled Page</title>\n";
  
  // Link to the CSS route
  html += "<link rel=\"stylesheet\" type=\"text/css\" href=\"/style.css\">\n";
  
  html += "</head>\n<body>\n";
  html += "  <div class=\"card\">\n";
  html += "    <h1>External Stylesheet</h1>\n";
  html += "    <p>This web page's layout and colors are defined in a separate CSS file served from the ESP32. This keeps the HTML clean and improves page load times by allowing the browser to cache the CSS file.</p>\n";
  html += "    <span class=\"badge\">CSS Route Active</span>\n";
  html += "  </div>\n";
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// 3. CSS URL handler ("/style.css")
void handleCSS() {
  // Set cache control headers to tell the browser to cache the file
  server.sendHeader("Cache-Control", "max-age=86400"); // Cache for 1 day
  
  // Send the CSS payload with the correct text/css Content-Type
  server.send(200, "text/css", CSS_CONTENT);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\nESP32 CSS Web Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Register routes
  server.on("/", handleRoot);
  server.on("/style.css", handleCSS);
  
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
3. Open your web browser and navigate to the printed IP address. Verify that the styled page loads correctly.
4. Inspect the page source or network tab to confirm that `/style.css` is loaded independently.

## Expected Output
Serial Monitor:
```
ESP32 CSS Web Server Starting...
....
WiFi Connected.
Web Server active. Connect at: http://10.10.0.3
```

Browser Page Layout:
* A dark-themed card container with rounded corners and border styling, containing styled headers and text.

## Expected Canvas Behavior
* Navigating to the page requests the stylesheet, logging the connection in the background.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `<link rel="stylesheet" ...>` | Directs the browser to fetch the styling rules from the `/style.css` route. |
| `server.send(200, "text/css", ...)` | Serves the stylesheet with the correct MIME type so the browser can parse it. |
| `Cache-Control` | Adds caching headers to instruct the browser to store the stylesheet locally, reducing queries. |

## Hardware & Safety Concept: MIME Types and Browser Security
Web browsers enforce strict security policies. If a server returns a CSS file with an incorrect Content-Type header (such as `text/plain` or `text/html`), modern browsers will reject the stylesheet under strict MIME type checking rules. Enforcing the correct `text/css` MIME type ensures the styles render correctly.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the count of CSS requests.
2. **Dynamic Color Theme**: Read a potentiometer (Project 34) and dynamically adjust a color variable in the CSS stylesheet response.
3. **SPIFFS integration**: Store the CSS stylesheet in the flash memory file system (SPIFFS) and serve it from there (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The webpage loads with no styling | Content-Type header wrong | Verify that the `/style.css` response uses the `text/css` MIME type |
| CSS changes do not show up | Browser caching | Clear your browser cache or open the page in an incognito window |
| Server crashes on large stylesheet | Stack overflow | Store the stylesheet string in flash memory using `PROGMEM` |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [51 - ESP32 Single Page HTTP Web Server](51-esp32-single-page-http-web-server.md)
- [59 - ESP32 Web Server serving JavaScript scripts](59-esp32-web-server-serving-javascript-scripts.md) (Next project)
- [62 - ESP32 Web Server serving files from SPIFFS](62-esp32-web-server-serving-files-from-spiffs.md)
