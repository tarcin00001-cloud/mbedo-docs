# 52 - Web Page LED Toggle Button

Build a local web server on the ESP32 that hosts an interactive control page with a styled HTML button, processes toggling events via form-post actions, and switches an LED on GPIO 13.

## Goal
Learn how to handle interactive HTML forms, process HTTP POST requests, update and render output states, and redirect clients.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a web server on port 80. An LED is on GPIO 13. Navigating to the ESP32's IP address displays a webpage containing a button. Clicking the button sends an HTTP POST request to the server, which toggles the state of the LED and refreshes the webpage to display the updated state.

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
// Web Page LED Toggle Button (Interactive HTML form handling)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int LED_PIN = 13;

WebServer server(80);

// Root URL Route Handler ("/")
void handleRoot() {
  bool ledState = (digitalRead(LED_PIN) == HIGH);
  
  // 1. Compile HTML page with a styled POST action form
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>ESP32 LED Controller</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 12px; padding: 30px; box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.5); max-width: 350px; width: 100%; text-align: center; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-bottom: 25px; }\n";
  html += "  .status { font-size: 18px; margin-bottom: 30px; color: #94a3b8; }\n";
  html += "  .status-val { font-weight: bold; color: " + String(ledState ? "#2ec4b6" : "#e71d36") + "; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 14px; font-size: 16px; font-weight: bold; color: white; border: none; border-radius: 6px; cursor: pointer; text-decoration: none; transition: background-color 0.2s; }\n";
  html += "  .btn-on { background-color: #059669; }\n";
  html += "  .btn-on:hover { background-color: #047857; }\n";
  html += "  .btn-off { background-color: #dc2626; }\n";
  html += "  .btn-off:hover { background-color: #b91c1c; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>LED Control Panel</h1>\n";
  html += "  <p class=\"status\">LED is currently: <span class=\"status-val\">" + String(ledState ? "ON" : "OFF") + "</span></p>\n";
  
  // HTML form targeting /toggle via POST request method
  html += "  <form action=\"/toggle\" method=\"POST\">\n";
  if (ledState) {
    html += "    <button type=\"submit\" class=\"btn btn-off\">Turn LED OFF</button>\n";
  } else {
    html += "    <button type=\"submit\" class=\"btn btn-on\">Turn LED ON</button>\n";
  }
  html += "  </form>\n";
  
  html += "</div>\n";
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// POST Toggle Handler ("/toggle")
void handleToggle() {
  // Verify request is POST
  if (server.method() == HTTP_POST) {
    bool currentState = digitalRead(LED_PIN);
    digitalWrite(LED_PIN, !currentState);
    Serial.printf("[Server] Toggle request processed. Next State: %s\n", !currentState ? "ON" : "OFF");
  }
  
  // Redirect back to root page to show updated state
  server.sendHeader("Location", "/");
  server.send(303);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW); // LED starts OFF
  
  Serial.println("\nESP32 LED Form Web Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Register routes
  server.on("/", handleRoot);
  server.on("/toggle", HTTP_POST, handleToggle); // Enforce POST method for this route
  
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
4. Open your web browser and navigate to the printed IP address. Click the toggle button to turn the LED ON and OFF.

## Expected Output
Serial Monitor:
```
ESP32 LED Form Web Server Starting...
....
WiFi Connected.
Web Server active. Connect at: http://10.10.0.3
[Server] Toggle request processed. Next State: ON
[Server] Toggle request processed. Next State: OFF
```

## Expected Canvas Behavior
* Clicking the toggle button on the web page switches the Red LED widget ON and OFF.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `HTTP_POST` | Enforces that only HTTP POST requests are processed by the toggle route handler. |
| `server.method() == HTTP_POST` | Validates that the incoming request uses the POST protocol method. |
| `server.send(303)` | Redirects the client browser back to `/` to refresh the dashboard status. |

## Hardware & Safety Concept: HTTP GET vs HTTP POST for Control Actions
* **HTTP GET**: Should only be used to request data, not to modify server state. If GET is used to toggle an LED (e.g. `/toggle`), search engine crawlers indexing the page or browser pre-fetching configurations could trigger the action automatically, causing unexpected behavior.
* **HTTP POST**: Transmits parameters in the request body. This is the correct method for state-changing control actions.

## Try This! (Challenges)
1. **Interactive OLED Status**: Add an OLED screen (Project 60) and display the current LED state.
2. **Double Button Panel**: Modify the form to display two separate buttons (ON and OFF) instead of a single toggle button.
3. **Buzzer Confirmation**: Sound a quick beep on a buzzer (GPIO 4) when the state changes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The webpage fails to load | Client on different network | Ensure your computer and the ESP32 are connected to the same WiFi network |
| Toggle button has no effect | HTTP method mismatch | Confirm that the form method is set to `POST` and matches the route configuration |
| The ESP32 hangs | Blocking calls | Avoid using blocking `delay()` calls inside the main loop |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [44 - ESP32 Web-based LED toggle](../beginner/44-esp32-web-based-led-toggle.md)
- [51 - ESP32 Single Page HTTP Web Server](51-esp32-single-page-http-web-server.md)
- [53 - ESP32 Web Page Multi-LED Control panel](53-esp32-web-page-multi-led-control-panel.md) (Next project)
