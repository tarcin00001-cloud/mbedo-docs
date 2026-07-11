# 44 - ESP32 Web-Based LED Toggle

Build a local web server on the ESP32 that hosts a control page, listens for URL parameter triggers (`/H` and `/L`), and toggles a physical LED connected to GPIO 13.

## Goal
Learn how to use the `WebServer` library, configure URL route handlers, serve HTML text payloads, and control outputs via HTTP requests.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a web server on port 80. An LED is on GPIO 13. Navigating to the ESP32's IP address (e.g. `http://10.10.0.3/`) in a web browser displays a page with buttons to control the LED. Navigating to `/H` turns the LED ON, and `/L` turns it OFF.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Green LED | Anode (+) | GPIO13 via 330 Ω | Green | Web-controlled LED |
| Shared ground | GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the Green LED anode to GPIO 13 via a 330 Ω resistor.

## Code
```cpp
// Web-based LED toggle (Simple Web Server route handler)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int LED_PIN = 13;

// Host Web Server on standard HTTP Port 80
WebServer server(80);

// 1. Root URL handler ("/")
void handleRoot() {
  String html = "<html><head><title>ESP32 Web LED</title>";
  html += "<style>body { font-family: sans-serif; text-align: center; margin-top: 50px; }";
  html += "a { display: inline-block; padding: 15px 25px; margin: 10px; font-size: 20px; color: white; text-decoration: none; border-radius: 5px; }";
  html += ".on { background-color: #2ec4b6; } .off { background-color: #e71d36; }</style></head>";
  html += "<body><h1>ESP32 Web-Controlled LED</h1>";
  html += "<p>Current LED State: <strong>";
  html += digitalRead(LED_PIN) == HIGH ? "ON" : "OFF";
  html += "</strong></p>";
  html += "<a href=\"/H\" class=\"on\">Turn LED ON</a>";
  html += "<a href=\"/L\" class=\"off\">Turn LED OFF</a>";
  html += "</body></html>";
  
  server.send(200, "text/html", html);
}

// 2. LED ON URL handler ("/H")
void handleLEDOn() {
  digitalWrite(LED_PIN, HIGH);
  Serial.println("[Web Action] LED turned ON via web client request.");
  
  // Send status page or redirect back to root
  // 303 Redirect forces browser to load "/"
  server.sendHeader("Location", "/");
  server.send(303); 
}

// 3. LED OFF URL handler ("/L")
void handleLEDOff() {
  digitalWrite(LED_PIN, LOW);
  Serial.println("[Web Action] LED turned OFF via web client request.");
  
  server.sendHeader("Location", "/");
  server.send(303);
}

// 4. Custom 404 page handler
void handleNotFound() {
  server.send(404, "text/plain", "Error 404: Path Not Found on ESP32 Web Server.");
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW); // Start with LED OFF
  
  Serial.println("\n==================================");
  Serial.println("ESP32 Web Server LED Toggle Setup");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
  
  // 5. Register URL routes to handler functions
  server.on("/", handleRoot);
  server.on("/H", handleLEDOn);
  server.on("/L", handleLEDOff);
  server.onNotFound(handleNotFound);
  
  // 6. Start the Web Server
  server.begin();
  
  Serial.print("Web Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  Serial.println("==================================\n");
}

void loop() {
  // 7. Handle client HTTP requests non-blockingly
  server.handleClient();
  
  delay(2); // Yield to system
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Wire LED to **GPIO13**.
3. Paste the code and click **Run**.
4. Note the printed IP address (e.g. `http://10.10.0.3`).
5. Open your web browser and navigate to `http://10.10.0.3/H` to turn the LED ON, or `http://10.10.0.3/L` to turn it OFF.

## Expected Output
Serial Monitor:
```
==================================
ESP32 Web Server LED Toggle Setup
==================================
WiFi Connected successfully.
Web Server active. Connect at: http://10.10.0.3
==================================

[Web Action] LED turned ON via web client request.
[Web Action] LED turned OFF via web client request.
```

Web Browser screen shows:
* A dashboard displaying the current LED state with two buttons to turn the LED ON and OFF.

## Expected Canvas Behavior
* Clicking the buttons on the web page toggles the Green LED widget.
* The Serial Monitor logs the client requests.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WebServer server(80)` | Instantiates a web server object listening on standard HTTP port 80. |
| `server.on("/", ...)` | Binds URL route paths to their handler functions. |
| `server.send(...)` | Transmits the HTTP response (code 200, Content-Type, and HTML payload). |
| `server.handleClient()` | Processes incoming client requests in the background. |

## Hardware & Safety Concept: HTTP Redirects and Web Resource Design
When a user clicks a button on a web page, the browser sends an HTTP request to that URL (e.g. `/H`). If the server replies with a confirmation string, the browser loads a new page, leaving the control dashboard. Sending an **HTTP 303 Redirect header** redirects the browser back to the root page (`/`), updating the dashboard to reflect the new state.

## Try This! (Challenges)
1. **Interactive OLED Status**: Add an OLED screen (Project 60) and display the count of web requests.
2. **Web-controlled Buzzer**: Add a route `/B` to play a warning tone on a buzzer (GPIO 4).
3. **HTTP Authentication**: Add basic password authentication to the server, protecting the control page (Project 64).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The webpage fails to load | Client on different network | Ensure your computer and the ESP32 are connected to the same WiFi network |
| LED does not toggle | Pin assignment conflict | Verify that the LED is connected to GPIO 13 |
| Server is slow to respond | Blocking calls in loop | Remove any blocking `delay()` calls inside the main loop |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [45 - ESP32 Web-based Buzzer chime trigger](45-esp32-web-based-buzzer-chime-trigger.md) (Next project)
- [46 - ESP32 Web-based Relay switch control](46-esp32-web-based-relay-switch-control.md)
- [51 - ESP32 Single Page HTTP Web Server](../../intermediate/51-esp32-single-page-http-web-server.md)
