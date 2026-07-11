# 46 - ESP32 Web-Based Relay Switch Control

Build a local home automation web server on the ESP32 that hosts a control page, listens for URL parameter triggers (`/toggle`), drives a relay connected to GPIO 13 to control connected loads, and redirects the browser to refresh the status.

## Goal
Learn how to configure URL route handlers to actuate relays, serve HTML pages, manage redirect headers, and build web control dashboards.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a web server on port 80. A relay is on GPIO 13. Navigating to the ESP32's IP address in a web browser displays a page with the current relay state and a button to toggle the switch. Clicking the button toggles the relay contacts and updates the page.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Relay Module | IN (Signal) | GPIO13 | Orange | Load controller switch |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the relay from the 5V Vin rail.

## Code
```cpp
// Web-based Relay switch control (Web Server Home Automation)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int RELAY_PIN = 13;

WebServer server(80);

// 1. Root URL handler ("/")
void handleRoot() {
  bool relayState = (digitalRead(RELAY_PIN) == HIGH);
  
  String html = "<html><head><title>ESP32 Web Relay</title>";
  html += "<style>body { font-family: sans-serif; text-align: center; margin-top: 50px; }";
  html += ".btn { display: inline-block; padding: 15px 30px; margin: 20px; font-size: 20px; color: white; text-decoration: none; border-radius: 5px; }";
  html += ".on { background-color: #2ec4b6; } .off { background-color: #e71d36; }";
  html += ".status { font-size: 24px; font-weight: bold; color: ";
  html += relayState ? "#2ec4b6" : "#e71d36";
  html += "; }</style></head>";
  html += "<body><h1>ESP32 Web-Controlled Relay</h1>";
  html += "<p>Relay Status: <span class=\"status\">";
  html += relayState ? "CLOSED (ON)" : "OPEN (OFF)";
  html += "</span></p>";
  html += "<a href=\"/toggle\" class=\"btn ";
  html += relayState ? "off" : "on";
  html += "\">";
  html += relayState ? "Turn LOAD OFF" : "Turn LOAD ON";
  html += "</a>";
  html += "</body></html>";
  
  server.send(200, "text/html", html);
}

// 2. Toggle URL handler ("/toggle")
void handleToggle() {
  bool nextState = !digitalRead(RELAY_PIN);
  digitalWrite(RELAY_PIN, nextState ? HIGH : LOW);
  
  Serial.print("[Web Action] Relay contacts toggled. State: ");
  Serial.println(nextState ? "CLOSED (ON)" : "OPEN (OFF)");
  
  // 3. Send 303 Redirect header back to root
  // Forces the browser to load "/" to refresh the dashboard status
  server.sendHeader("Location", "/");
  server.send(303);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start with relay OPEN (off)
  
  Serial.println("\n==================================");
  Serial.println("ESP32 Web Relay Switch Setup");
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
  server.on("/toggle", handleToggle);
  
  // Start server
  server.begin();
  
  Serial.print("Web Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  Serial.println("==================================\n");
}

void loop() {
  server.handleClient();
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Relay** onto the canvas.
2. Wire Relay to **GPIO13**.
3. Paste the code and click **Run**.
4. Note the printed IP address (e.g. `http://10.10.0.3`).
5. Open your web browser and navigate to the IP. Click the button to toggle the relay.

## Expected Output
Serial Monitor:
```
==================================
ESP32 Web Relay Switch Setup
==================================
WiFi Connected successfully.
Web Server active. Connect at: http://10.10.0.3
==================================

[Web Action] Relay contacts toggled. State: CLOSED (ON)
[Web Action] Relay contacts toggled. State: OPEN (OFF)
```

Web Browser screen shows:
* A dashboard displaying the current relay state with a button to toggle the switch.

## Expected Canvas Behavior
* Clicking the button on the web page toggles the relay widget ON (green) and OFF (grey).
* The Serial Monitor logs the client requests.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WebServer server(80)` | Instantiates a web server object listening on standard HTTP port 80. |
| `server.on("/toggle", ...)` | Binds URL route paths to their handler functions. |
| `digitalWrite(RELAY_PIN, ...)` | Controls the relay contacts to switch the connected load. |
| `server.send(303)` | Redirects the browser back to the root page (`/`) to refresh the dashboard status. |

## Hardware & Safety Concept: Electrical Isolation and Inductive Snubbing
Relays isolate the low-voltage microcontroller (3.3V) from high-voltage AC loads (such as mains power). When a relay switches inductive loads (like motors or transformers), a high-voltage spike is generated across the contacts (back EMF), which can cause arc discharges that melt the contacts. To protect the relay, install an external RC snubber circuit or varistor in parallel with the load.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the active relay status.
2. **Timed Auto-Off**: Add a route `/pulse?duration=5000` to turn the relay ON for 5 seconds and then turn it off automatically.
3. **Buzzer alert**: Sound a quick beep on a buzzer (GPIO 4) when the relay state changes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The webpage fails to load | Client on different network | Ensure your computer and the ESP32 are connected to the same WiFi network |
| Relay does not switch | Pin assignment conflict | Verify that the relay signal line is wired to GPIO 13 |
| Relay clicks continuously | Loop timing issue | Avoid using blocking `delay()` calls inside the main loop |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [44 - ESP32 Web-based LED toggle](44-esp32-web-based-led-toggle.md)
- [45 - ESP32 Web-based Buzzer chime trigger](45-esp32-web-based-buzzer-chime-trigger.md)
- [21 - ESP32 UDP Control Switch](21-esp32-udp-control-switch.md)
