# 60 - Web Server AJAX LED Toggle (no page refresh)

Build a modern, single-page web control panel on the ESP32 that uses client-side JavaScript (AJAX) to send background HTTP requests to a `/api/toggle` endpoint, toggles an LED on GPIO 13, and updates the webpage state dynamically without reloading the page.

## Goal
Learn how to use AJAX (Asynchronous JavaScript and XML) via the `fetch()` API, parse JSON payloads on the client side, serve JSON responses from the server, and design responsive user interfaces.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a web server on port 80. An LED is on GPIO 13. Navigating to the ESP32's IP address displays a control page with a button and a status indicator light. Clicking the button sends a background request to the server, which toggles the LED and returns the updated state as JSON. The page JavaScript updates the status indicator immediately without reloading the page.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Red LED | Anode (+) | GPIO13 via 330 Ω | Red | Controlled output LED |
| Shared ground | GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the Red LED anode to GPIO 13 via a 330 Ω resistor.

## Code
```cpp
// Web Server AJAX LED toggle (Asynchronous control panel)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int LED_PIN = 13;

WebServer server(80);

// 1. Root URL Handler ("/")
// Serves the HTML, CSS, and inline AJAX Javascript code
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>ESP32 AJAX LED Controller</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 320px; width: 100%; text-align: center; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; margin-bottom: 25px; }\n";
  
  // Status indicator light (circle)
  html += "  .indicator-container { display: flex; justify-content: center; align-items: center; margin-bottom: 30px; }\n";
  html += "  .indicator { height: 40px; width: 40px; border-radius: 50%; background-color: #475569; transition: background-color 0.3s; margin-right: 15px; box-shadow: 0 0 10px rgba(0,0,0,0.5); }\n";
  html += "  .indicator.on { background-color: #f43f5e; box-shadow: 0 0 20px #f43f5e; }\n";
  html += "  .status-text { font-size: 18px; font-weight: bold; color: #94a3b8; }\n";
  html += "  .status-text.on { color: #f43f5e; }\n";
  
  html += "  .btn { display: inline-block; width: 100%; padding: 14px; font-size: 16px; font-weight: bold; color: white; background-color: #38bdf8; border: none; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; }\n";
  html += "  .btn:hover { background-color: #0ea5e9; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>AJAX Controller</h1>\n";
  
  // Indicator status block
  html += "  <div class=\"indicator-container\">\n";
  html += "    <div id=\"ledIndicator\" class=\"indicator\"></div>\n";
  html += "    <span id=\"statusLabel\" class=\"status-text\">OFF</span>\n";
  html += "  </div>\n";
  
  html += "  <button class=\"btn\" onclick=\"toggleLED()\">Toggle LED State</button>\n";
  html += "</div>\n";
  
  // 2. Client-side AJAX script using Fetch API
  html += "<script>\n";
  html += "  // Query current state on load\n";
  html += "  window.onload = function() {\n";
  html += "    fetch('/api/state')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => updateUI(data.state));\n";
  html += "  };\n";
  
  html += "  function toggleLED() {\n";
  html += "    fetch('/api/toggle', { method: 'POST' })\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => updateUI(data.state))\n";
  html += "      .catch(err => console.error('Error toggling LED:', err));\n";
  html += "  }\n";
  
  html += "  function updateUI(state) {\n";
  html += "    const ind = document.getElementById('ledIndicator');\n";
  html += "    const lbl = document.getElementById('statusLabel');\n";
  html += "    if (state === 1) {\n";
  html += "      ind.classList.add('on');\n";
  html += "      lbl.classList.add('on');\n";
  html += "      lbl.innerText = 'ON';\n";
  html += "    } else {\n";
  html += "      ind.classList.remove('on');\n";
  html += "      lbl.classList.remove('on');\n";
  html += "      lbl.innerText = 'OFF';\n";
  html += "    }\n";
  html += "  }\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// 3. API Query State Handler ("/api/state")
void handleStateAPI() {
  bool ledState = (digitalRead(LED_PIN) == HIGH);
  String json = "{\"state\":" + String(ledState ? 1 : 0) + "}";
  
  server.send(200, "application/json", json);
}

// 4. API Toggle Command Handler ("/api/toggle")
void handleToggleAPI() {
  bool nextState = !digitalRead(LED_PIN);
  digitalWrite(LED_PIN, nextState ? HIGH : LOW);
  
  Serial.printf("[Server] Asynchronous toggle processed. State -> %s\n", nextState ? "ON" : "OFF");
  
  String json = "{\"state\":" + String(nextState ? 1 : 0) + "}";
  server.send(200, "application/json", json);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW); // Start OFF
  
  Serial.println("\nESP32 AJAX Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Register routes
  server.on("/", handleRoot);
  server.on("/api/state", handleStateAPI);
  server.on("/api/toggle", HTTP_POST, handleToggleAPI); // POST toggle API route
  
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
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Wire LED to **GPIO13**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Click the toggle button. Observe that the LED toggles and the dashboard updates without reloading.

## Expected Output
Serial Monitor:
```
ESP32 AJAX Server Starting...
....
WiFi Connected.
Web Server active. Connect at: http://10.10.0.3
[Server] Asynchronous toggle processed. State -> ON
[Server] Asynchronous toggle processed. State -> OFF
```

Browser Console (Inspect Network tab):
* Triggering the button shows lightweight POST requests to `/api/toggle` returning JSON strings (`{"state":1}`).

## Expected Canvas Behavior
* Clicking the button on the webpage immediately toggles the Red LED widget, with no browser page reload.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `fetch('/api/toggle', ...)` | Sends a background POST request to the server without interfering with the active webpage. |
| `response.json()` | Parses the returned JSON string into a client-side JavaScript object. |
| `updateUI(...)` | Modifies the CSS classes of DOM elements to update colors and labels. |
| `server.send(200, "application/json", ...)` | Serves the state payload with the correct JSON MIME type. |

## Hardware & Safety Concept: Asynchronous (AJAX) Control vs Traditional Reloads
* **Traditional Reloads**: Requesting an action (such as `/H` or `/L`) requires compiling and transmitting the entire HTML document (several kilobytes). This flashes the screen, consumes CPU cycles on the ESP32, and increases latency.
* **AJAX (Asynchronous Requests)**: Requests only raw data payloads (a few bytes of JSON). The browser processes the response in the background and updates only the modified elements, reducing CPU load and lag.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a count of AJAX commands received.
2. **Web-controlled Buzzer**: Add an AJAX button to trigger a buzzer chime (Project 45).
3. **RGB Web Controller**: Connect an RGB LED (Project 44) and add three sliders to adjust color channels asynchronously.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Clicking the button has no effect | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |
| API requests return 404 | Route not registered | Verify that `/api/toggle` is registered in the setup function |
| Status indicator stays grey | JSON parse error | Verify the server sends valid JSON strings (e.g. quotes around keys, correct brackets) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [52 - ESP32 Web Page LED Toggle button](52-esp32-web-page-led-toggle-button.md)
- [59 - ESP32 Web Server serving JavaScript scripts](59-esp32-web-server-serving-javascript-scripts.md)
- [61 - ESP32 Web Server AJAX Telemetry Auto-refresh](61-esp32-web-server-ajax-telemetry-auto-refresh.md) (Next project)
