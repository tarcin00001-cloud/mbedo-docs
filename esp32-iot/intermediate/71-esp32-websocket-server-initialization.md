# 71 - WebSocket Server initialization

Configure a persistent, full-duplex WebSocket server on the ESP32 that listens on port 81, accepts incoming client connection handshakes, processes text-based messages, and logs connection events to the Serial Monitor.

## Goal
Learn how to initialize the `WebSocketsServer` library, register event callback handlers, capture client connection and disconnection states, and send and receive text payloads.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It hosts an HTTP server on port 80 (to serve a basic webpage containing JavaScript) and a WebSocket server on port 81. Navigating to the webpage establishes a persistent WebSocket connection, logging the handshake on the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// WebSocket Server initialization (Full-duplex socket handshake logger)
#include <WiFi.h>
#include <WebServer.h>
#include <WebSocketsServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

WebServer server(80);
WebSocketsServer webSocket = WebSocketsServer(81); // WebSocket server on port 81

// 1. WebSocket Event Callback Handler
// Processes all incoming client events (connections, text data, disconnections)
void webSocketEvent(uint8_t num, WStype_t type, uint8_t * payload, size_t length) {
  switch (type) {
    case WStype_DISCONNECTED:
      Serial.printf("[WebSocket] Client #%u disconnected.\n", num);
      break;
      
    case WStype_CONNECTED: {
      IPAddress ip = webSocket.remoteIP(num);
      Serial.printf("[WebSocket] Client #%u connected from IP: %d.%d.%d.%d\n", 
                    num, ip[0], ip[1], ip[2], ip[3]);
                    
      // Send welcome text message to the newly connected client
      webSocket.sendTXT(num, "SYSTEM: Connected to ESP32 WebSocket server.");
      break;
    }
    
    case WStype_TEXT:
      Serial.printf("[WebSocket] Message from #%u: %s\n", num, payload);
      
      // Echo the message back to the sender client
      String response = "ECHO: " + String((char*)payload);
      webSocket.sendTXT(num, response.c_str());
      break;
  }
}

// 2. Root URL Handler ("/")
// Serves the HTML page containing JavaScript to open the WebSocket channel
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>ESP32 WebSocket Test</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 12px; padding: 30px; box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.4); max-width: 400px; width: 100%; text-align: center; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-bottom: 15px; }\n";
  html += "  #log { height: 150px; background-color: #0f172a; border-radius: 6px; padding: 10px; overflow-y: auto; text-align: left; font-family: monospace; font-size: 13px; color: #10b981; border: 1px solid #334155; margin-bottom: 20px; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 12px; font-size: 15px; font-weight: bold; color: white; background-color: #38bdf8; border: none; border-radius: 6px; cursor: pointer; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>WebSocket Console</h1>\n";
  html += "  <div id=\"log\">Initializing connection...</div>\n";
  html += "  <button class=\"btn\" onclick=\"sendMessage()\">Send Test Message</button>\n";
  html += "</div>\n";
  
  // 3. Client-side WebSocket script
  html += "<script>\n";
  // Establish WebSocket connection to port 81 of the host IP
  html += "  const ws = new WebSocket('ws://' + window.location.hostname + ':81/');\n";
  html += "  const logEl = document.getElementById('log');\n";
  
  html += "  ws.onopen = function() {\n";
  html += "    logEl.innerHTML = 'Status: Connected to WebSocketServer<br>';\n";
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
  html += "    ws.send(msg); // Send text message over WebSocket\n";
  html += "  }\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\nESP32 WebSocket Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
  
  // Start HTTP Server
  server.on("/", handleRoot);
  server.begin();
  
  // 4. Start WebSocket Server
  webSocket.begin();
  webSocket.onEvent(webSocketEvent); // Register event callback
  
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  Serial.printf("WebSocket Server active on port 81.\n");
}

void loop() {
  server.handleClient();
  
  // 5. Yield background processing to the WebSocket engine
  webSocket.loop();
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Open your web browser and navigate to the printed IP address.
4. Verify that the WebSocket console shows "Status: Connected".
5. Click the "Send Test Message" button. Watch the message transmit and echo back immediately.

## Expected Output
Serial Monitor:
```
ESP32 WebSocket Server Starting...
....
WiFi Connected successfully.
HTTP Server active. Connect at: http://10.10.0.3
WebSocket Server active on port 81.

[WebSocket] Client #0 connected from IP: 10.10.0.5
[WebSocket] Message from #0: PING_42
```

Browser Console:
```
Status: Connected to WebSocketServer
Received -> SYSTEM: Connected to ESP32 WebSocket server.
Sent -> PING_42
Received -> ECHO: PING_42
```

## Expected Canvas Behavior
* Establishing connections and sending messages logs events immediately on the Serial Monitor.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WebSocketsServer(81)` | Instantiates a WebSocket server object listening on port 81. |
| `webSocket.onEvent(...)` | Registers the callback function to handle incoming events. |
| `new WebSocket('ws://' + ...)` | Client-side JavaScript API to open a persistent connection. |
| `webSocket.loop()` | Processes background network frames for active connections. |

## Hardware & Safety Concept: HTTP vs WebSockets in IoT
* **HTTP**: A request-response protocol. To receive data, the client must poll the server (e.g. once every second). This creates latency and generates significant packet overhead (HTTP headers are several hundred bytes).
* **WebSockets**: A persistent TCP channel. Both the client and server can send data at any time with minimal latency and header overhead (only 2 to 14 bytes per frame), making it ideal for real-time control.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a count of active WebSocket clients.
2. **Broadcast message**: Program the server to send an uptime packet to all connected clients every 5 seconds.
3. **Buzzer confirmation tone**: Sound a quick beep on a buzzer (GPIO 15) when a client connects.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Webpage loads but connection fails | Port blocked | Confirm that the WebSocket server is running on port 81 and not blocked by a firewall |
| Handshake times out | Memory exhaustion | Close other active sockets to free RAM for the connection |
| Connection drops | loop function blocked | Ensure `webSocket.loop()` is called continuously in the loop function without blocking delays |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [51 - ESP32 Single Page HTTP Web Server](51-esp32-single-page-http-web-server.md)
- [72 - ESP32 WebSocket LED Toggle (low latency feedback)](72-esp32-websocket-led-toggle-low-latency-feedback.md) (Next project)
- [73 - ESP32 WebSocket Sensor Streamer](73-esp32-websocket-sensor-streamer.md)
