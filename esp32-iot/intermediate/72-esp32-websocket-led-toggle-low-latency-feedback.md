# 72 - WebSocket LED Toggle (low latency feedback)

Build a high-performance web controller on the ESP32 that establishes a full-duplex WebSocket channel, processes sub-millisecond toggle commands ("TOGGLE") sent by client browsers, switches an LED on GPIO 13, and broadcasts state updates to all connected subscribers in real time.

## Goal
Learn how to implement low-latency WebSocket control links, broadcast state updates to multiple subscribers, process incoming text payloads, and design responsive user interfaces.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It hosts a webpage on port 80 and a WebSocket server on port 81. An LED is on GPIO 13. Navigating to the page opens a WebSocket channel. Clicking the toggle button sends a `TOGGLE` command, switching the LED. The ESP32 broadcasts the new state (`ON` or `OFF`) to all connected browser windows immediately, updating them in sync.

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
// WebSocket LED Toggle (Low-latency full-duplex multi-client control)
#include <WiFi.h>
#include <WebServer.h>
#include <WebSocketsServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int LED_PIN = 13;

WebServer server(80);
WebSocketsServer webSocket = WebSocketsServer(81);

// Helper to compile state payload
String getLEDStateString() {
  return (digitalRead(LED_PIN) == HIGH) ? "ON" : "OFF";
}

// WebSocket Event Callback Handler
void webSocketEvent(uint8_t num, WStype_t type, uint8_t * payload, size_t length) {
  switch (type) {
    case WStype_DISCONNECTED:
      Serial.printf("[WebSocket] Subscriber #%u left.\n", num);
      break;
      
    case WStype_CONNECTED:
      Serial.printf("[WebSocket] Subscriber #%u joined.\n", num);
      // Send active state immediately to sync the client UI
      webSocket.sendTXT(num, getLEDStateString().c_str());
      break;
      
    case WStype_TEXT:
      String cmd = String((char*)payload);
      cmd.trim();
      
      if (cmd == "TOGGLE") {
        // Toggle LED state
        bool nextState = !digitalRead(LED_PIN);
        digitalWrite(LED_PIN, nextState ? HIGH : LOW);
        
        Serial.printf("[Action] Toggle from client #%u. State: %s\n", num, nextState ? "ON" : "OFF");
        
        // Broadcast the new state to ALL connected clients immediately
        // This updates all browser windows in sync
        webSocket.broadcastTXT(getLEDStateString().c_str());
      }
      break;
  }
}

// Root URL Handler ("/")
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>WebSocket LED Controller</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 320px; width: 100%; text-align: center; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; margin-bottom: 25px; }\n";
  
  html += "  .indicator-container { display: flex; justify-content: center; align-items: center; margin-bottom: 30px; }\n";
  html += "  .indicator { height: 40px; width: 40px; border-radius: 50%; background-color: #475569; transition: background-color 0.2s, box-shadow 0.2s; margin-right: 15px; }\n";
  html += "  .indicator.on { background-color: #ef4444; box-shadow: 0 0 20px #ef4444; }\n";
  html += "  .status-text { font-size: 18px; font-weight: bold; color: #94a3b8; }\n";
  html += "  .status-text.on { color: #ef4444; }\n";
  
  html += "  .btn { display: inline-block; width: 100%; padding: 14px; font-size: 16px; font-weight: bold; color: white; background-color: #38bdf8; border: none; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; }\n";
  html += "  .btn:hover { background-color: #0ea5e9; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Live LED Sync</h1>\n";
  
  // Status indicator
  html += "  <div class=\"indicator-container\">\n";
  html += "    <div id=\"ledIndicator\" class=\"indicator\"></div>\n";
  html += "    <span id=\"statusLabel\" class=\"status-text\">OFF</span>\n";
  html += "  </div>\n";
  
  html += "  <button class=\"btn\" onclick=\"toggleLED()\">Toggle LED</button>\n";
  html += "</div>\n";
  
  // Client-side WebSocket script
  html += "<script>\n";
  html += "  const ws = new WebSocket('ws://' + window.location.hostname + ':81/');\n";
  
  html += "  ws.onmessage = function(evt) {\n";
  html += "    const ind = document.getElementById('ledIndicator');\n";
  html += "    const lbl = document.getElementById('statusLabel');\n";
  
  html += "    if (evt.data === 'ON') {\n";
  html += "      ind.classList.add('on');\n";
  html += "      lbl.classList.add('on');\n";
  html += "      lbl.innerText = 'ON';\n";
  html += "    } else {\n";
  html += "      ind.classList.remove('on');\n";
  html += "      lbl.classList.remove('on');\n";
  html += "      lbl.innerText = 'OFF';\n";
  html += "    }\n";
  html += "  };\n";
  
  html += "  function toggleLED() {\n";
  html += "    ws.send('TOGGLE'); // Send command over open socket\n";
  html += "  }\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW); // Start OFF
  
  Serial.println("\nESP32 WebSocket Control Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
  
  // Register routes
  server.on("/", handleRoot);
  server.begin();
  
  // Start WebSocket Server
  webSocket.begin();
  webSocket.onEvent(webSocketEvent);
  
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  webSocket.loop();
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Wire LED to **GPIO13**.
3. Paste the code and click **Run**.
4. Open two separate web browser windows and navigate to the printed IP address in both.
5. Click the toggle button in one window. Verify that the LED on the canvas and the indicators in **both** browser windows update instantly.

## Expected Output
Serial Monitor:
```
ESP32 WebSocket Control Server Starting...
....
WiFi Connected successfully.
HTTP Server active. Connect at: http://10.10.0.3

[WebSocket] Subscriber #0 joined.
[WebSocket] Subscriber #1 joined.
[Action] Toggle from client #0. State: ON
[Action] Toggle from client #1. State: OFF
```

## Expected Canvas Behavior
* Clicking the button in either browser window immediately toggles the Red LED widget, with no browser page reload.
* The state of the indicators in both browsers updates in sync.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `webSocket.broadcastTXT(...)` | Transmits a text message to all connected clients simultaneously. |
| `ws.send('TOGGLE')` | Client-side command to send the message over the active socket channel. |
| `ws.onmessage` | Client-side callback triggered when the browser receives a message from the server. |

## Hardware & Safety Concept: WebSocket Multi-Client Synchronization
In home automation systems, multiple control interfaces (such as wall panels, smartphone apps, and tablet displays) can view the same device. If the server does not synchronize state changes across all clients, interfaces will display conflicting data. Enforcing **broadcasting** ensures all active displays reflect the actual hardware state instantly.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a checklist showing which clients are active.
2. **Double channel switch**: Add a second LED on GPIO 12 and allow controlling both independently using WebSockets.
3. **Buzzer confirmation tone**: Sound a quick beep on a buzzer (GPIO 15) when the LED switches.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Webpage loads but buttons do not toggle | connection failed | Open the browser's developer console (F12) to inspect JavaScript console errors |
| Toggles are laggy or drop | loop function blocked | Ensure `webSocket.loop()` is called continuously in the loop function without blocking delays |
| Browser does not receive state | broadcast missing | Confirm that `webSocket.broadcastTXT()` is called inside the message handler |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [60 - ESP32 Web Server AJAX LED toggle (no page refresh)](60-esp32-web-server-ajax-led-toggle-no-page-refresh.md)
- [71 - ESP32 WebSocket Server initialization](71-esp32-websocket-server-initialization.md)
- [73 - ESP32 WebSocket Sensor Streamer](73-esp32-websocket-sensor-streamer.md) (Next project)
