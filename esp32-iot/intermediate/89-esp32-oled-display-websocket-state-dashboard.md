# 89 - OLED display WebSocket state dashboard

Build a WebSocket monitoring station on the ESP32 that hosts a control page, tracks active client connection counts, and displays the local IP address and subscriber count on an SSD1306 OLED display.

## Goal
Learn how to track WebSocket client statistics, display device diagnostics on SSD1306 OLED screens, and manage multi-client connection states.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An SSD1306 OLED display is on I2C (GPIO 21/22). The ESP32 hosts a webpage on port 80 and a WebSocket server on port 81. The OLED display shows the local IP address and the count of active WebSocket subscribers, updating instantly as clients join or leave the chat console.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| SSD1306 OLED Display (128x64 I2C) | `oled_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| OLED Display | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the I2C bus. Power the OLED from the 3.3V rail.

## Code
```cpp
// OLED display WebSocket state dashboard (Live Client Monitor Node)
#include <WiFi.h>
#include <WebServer.h>
#include <WebSocketsServer.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

const char* ssid = "Wokwi-GUEST";
const char* password = "";

WebServer server(80);
WebSocketsServer webSocket = WebSocketsServer(81);

// Update diagnostic panel on the OLED screen
void updateOLEDDisplay() {
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  
  // 1. Draw Title Header
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.println("WS STATE DASHBOARD");
  display.drawFastHLine(0, 10, 128, SSD1306_WHITE);
  
  // 2. Display connection state and local IP
  display.setCursor(0, 18);
  if (WiFi.status() == WL_CONNECTED) {
    display.print("IP: ");
    display.println(WiFi.localIP());
  } else {
    display.println("WiFi: DISCONNECTED");
  }
  
  // 3. Display active WebSocket client statistics
  int clientCount = webSocket.connectedClients();
  display.setCursor(0, 34);
  display.print("Clients: ");
  display.setTextSize(2);
  display.print(clientCount);
  
  display.setTextSize(1);
  display.setCursor(0, 52);
  display.print("Server port: 81");
  
  display.display();
}

// WebSocket Event Callback Handler
void webSocketEvent(uint8_t num, WStype_t type, uint8_t * payload, size_t length) {
  switch (type) {
    case WStype_DISCONNECTED:
      Serial.printf("[WebSocket] Client #%u disconnected.\n", num);
      // Trigger OLED update immediately on disconnect
      updateOLEDDisplay();
      break;
      
    case WStype_CONNECTED:
      Serial.printf("[WebSocket] Client #%u connected.\n", num);
      // Trigger OLED update immediately on connection
      updateOLEDDisplay();
      break;
      
    case WStype_TEXT:
      // Echo message back to the sender client
      String response = "ECHO: " + String((char*)payload);
      webSocket.sendTXT(num, response.c_str());
      break;
  }
}

// Root URL Handler ("/")
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>OLED WebSocket Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 12px; padding: 30px; box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.4); max-width: 400px; width: 100%; text-align: center; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-bottom: 15px; }\n";
  html += "  #log { height: 150px; background-color: #0f172a; border-radius: 6px; padding: 10px; overflow-y: auto; text-align: left; font-family: monospace; font-size: 13px; color: #10b981; border: 1px solid #334155; margin-bottom: 20px; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 12px; font-size: 15px; font-weight: bold; color: white; background-color: #38bdf8; border: none; border-radius: 6px; cursor: pointer; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>OLED WS Console</h1>\n";
  html += "  <div id=\"log\">Connecting...</div>\n";
  html += "  <button class=\"btn\" onclick=\"sendMessage()\">Send ping</button>\n";
  html += "</div>\n";
  
  html += "<script>\n";
  html += "  const ws = new WebSocket('ws://' + window.location.hostname + ':81/');\n";
  html += "  const logEl = document.getElementById('log');\n";
  
  html += "  ws.onopen = function() {\n";
  html += "    logEl.innerHTML = 'Status: Connected to Server<br>';\n";
  html += "  };\n";
  
  html += "  ws.onmessage = function(evt) {\n";
  html += "    logEl.innerHTML += 'Received -> ' + evt.data + '<br>';\n";
  html += "    logEl.scrollTop = logEl.scrollHeight; // Auto scroll\n";
  html += "  };\n";
  
  html += "  ws.onclose = function() {\n";
  html += "    logEl.innerHTML += 'Status: Disconnected!<br>';\n";
  html += "  };\n";
  
  html += "  function sendMessage() {\n";
  html += "    const msg = 'PING_' + Math.floor(Math.random() * 100);\n";
  html += "    logEl.innerHTML += 'Sent -> ' + msg + '<br>';\n";
  html += "    ws.send(msg);\n";
  html += "  }\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Initialize SSD1306 OLED display
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println(R"(SSD1306 allocation failed)");
    for (;;);
  }
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.println("Initializing WiFi...");
  display.display();
  
  Serial.println("\nESP32 WebSocket OLED Monitor Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Start HTTP Server
  server.on("/", handleRoot);
  server.begin();
  
  // Start WebSocket Server
  webSocket.begin();
  webSocket.onEvent(webSocketEvent);
  
  // Initial print to OLED
  updateOLEDDisplay();
  
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  webSocket.loop();
  
  // Refresh display periodically to update connection states if they drop
  static unsigned long lastUpdate = 0;
  if (millis() - lastUpdate >= 1000) {
    updateOLEDDisplay();
    lastUpdate = millis();
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **SSD1306 OLED** onto the canvas.
2. Wire OLED to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Open several web browser windows and navigate to the printed IP address in each. Watch the client count on the OLED screen update.

## Expected Output
Serial Monitor:
```
ESP32 WebSocket OLED Monitor Starting...
....
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3

[WebSocket] Client #0 connected.
[WebSocket] Client #1 connected.
[WebSocket] Client #0 disconnected.
```

OLED Display:
```
WS STATE DASHBOARD
IP: 10.10.0.3
Clients: 1
Server port: 81
```

## Expected Canvas Behavior
* Opening or closing browser windows updates the subscriber count displayed on the OLED widget.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `display.begin(...)` | Mounts and starts the graphics library for the SSD1306 controller. |
| `webSocket.connectedClients()` | Queries the active client count. |
| `updateOLEDDisplay()` | Renders the text blocks and shapes on the OLED buffer and sends it to the screen. |
| `webSocket.loop()` | Processes background frames to parse subscription events. |

## Hardware & Safety Concept: OLED Burn-In Prevention and Screen Savers
SSD1306 OLED displays are susceptible to permanent screen burn-in if static images (like static text headers) are displayed continuously. To prevent burn-in:
1. **Reduce Brightness**: Do not run the screen at maximum brightness.
2. **Implement Screen Savers**: Write code to dim or turn OFF the display after a timeout (e.g. 5 minutes) if no new client connections or message events occur.

## Try This! (Challenges)
1. **OLED Screen Saver**: Implement a screensaver that turns off the display after 30 seconds of inactivity.
2. **Scroll History**: Display the last received message payload on the bottom line of the OLED display.
3. **Buzzer alert on join**: Sound a quick beep on a buzzer (GPIO 15) when a client connects.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The OLED display remains blank | I2C address mismatch | Verify the I2C address in `display.begin()` matches your module (usually `0x3C`) |
| Client count does not update | loop function blocked | Verify that the loop function has no blocking delay calls |
| Characters are too large | Text size misconfigured | Use `display.setTextSize(1)` to fit text on the screen |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [36 - ESP32 OLED Digital Clock NTP synced](../beginner/36-esp32-oled-digital-clock-ntp-synced.md)
- [60 - ESP32 Web Server AJAX LED toggle (no page refresh)](60-esp32-web-server-ajax-led-toggle-no-page-refresh.md)
- [71 - ESP32 WebSocket Server initialization](71-esp32-websocket-server-initialization.md)
