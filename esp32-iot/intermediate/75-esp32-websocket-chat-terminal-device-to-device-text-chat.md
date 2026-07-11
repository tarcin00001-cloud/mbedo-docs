# 75 - WebSocket Chat Terminal (device-to-device text chat)

Build a local network chat terminal hub on the ESP32 that establishes a full-duplex WebSocket server on port 81, accepts multiple client connections, relays chat messages asynchronously to all connected subscribers, and logs conversation transcripts to the Serial Monitor.

## Goal
Learn how to manage multi-client broadcasting, process string text buffers, design dynamic chat logs in JavaScript, and build communication hubs.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It hosts a webpage on port 80 and a WebSocket server on port 81. Multiple computers/smartphones can connect to the ESP32's IP address. Typing a message in one browser console broadcasts the message immediately to all other connected browser windows, creating a local network chat room.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// WebSocket Chat Terminal (Multi-client relay text terminal hub)
#include <WiFi.h>
#include <WebServer.h>
#include <WebSocketsServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

WebServer server(80);
WebSocketsServer webSocket = WebSocketsServer(81);

// WebSocket Event Callback Handler
void webSocketEvent(uint8_t num, WStype_t type, uint8_t * payload, size_t length) {
  switch (type) {
    case WStype_DISCONNECTED: {
      Serial.printf("[Chat Alert] User #%u left the chat.\n", num);
      
      // Broadcast disconnect event to all remaining users
      String disconnectMsg = "SYSTEM: User #" + String(num) + " disconnected.";
      webSocket.broadcastTXT(disconnectMsg.c_str());
      break;
    }
    
    case WStype_CONNECTED: {
      Serial.printf("[Chat Alert] User #%u joined the chat.\n", num);
      
      // Broadcast join event to all users
      String joinMsg = "SYSTEM: User #" + String(num) + " joined the chat.";
      webSocket.broadcastTXT(joinMsg.c_str());
      
      // Send welcome message to the new user
      webSocket.sendTXT(num, "SYSTEM: Welcome to ESP32 Local Chat Room.");
      break;
    }
    
    case WStype_TEXT: {
      String chatText = String((char*)payload);
      chatText.trim();
      
      if (chatText.length() > 0) {
        // Compile chat string format: "User #X: Hello!"
        String broadcastPayload = "User #" + String(num) + ": " + chatText;
        Serial.println(broadcastPayload);
        
        // Broadcast message to all active users
        webSocket.broadcastTXT(broadcastPayload.c_str());
      }
      break;
    }
  }
}

// Root URL Handler ("/")
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>ESP32 Chat Hub</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; display: flex; flex-direction: column; height: 80vh; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; margin-bottom: 20px; text-align: center; }\n";
  
  // Chat History Block
  html += "  #chatHistory { flex-grow: 1; background-color: #0f172a; border-radius: 8px; padding: 15px; overflow-y: auto; text-align: left; font-size: 14px; border: 1px solid #334155; margin-bottom: 20px; }\n";
  html += "  .msg-system { color: #64748b; font-style: italic; margin-bottom: 10px; }\n";
  html += "  .msg-user { color: #f1f5f9; margin-bottom: 10px; }\n";
  
  // Input elements
  html += "  .input-container { display: flex; gap: 10px; }\n";
  html += "  #messageInput { flex-grow: 1; padding: 12px; border-radius: 6px; border: 1px solid #334155; background-color: #0f172a; color: white; font-size: 14px; }\n";
  html += "  #messageInput:focus { outline: none; border-color: #38bdf8; }\n";
  html += "  .btn { padding: 12px 24px; font-size: 14px; font-weight: bold; color: white; background-color: #38bdf8; border: none; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; }\n";
  html += "  .btn:hover { background-color: #0ea5e9; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>ESP32 Local Chat</h1>\n";
  html += "  <div id=\"chatHistory\"></div>\n";
  
  html += "  <div class=\"input-container\">\n";
  html += "    <input type=\"text\" id=\"messageInput\" placeholder=\"Type message here...\" onkeypress=\"handleKeyPress(event)\">\n";
  html += "    <button class=\"btn\" onclick=\"sendChat()\">Send</button>\n";
  html += "  </div>\n";
  html += "</div>\n";
  
  // Client-side script
  html += "<script>\n";
  html += "  const history = document.getElementById('chatHistory');\n";
  html += "  const input = document.getElementById('messageInput');\n";
  
  html += "  const ws = new WebSocket('ws://' + window.location.hostname + ':81/');\n";
  
  html += "  ws.onmessage = function(evt) {\n";
  html += "    const isSystem = evt.data.startsWith('SYSTEM:');\n";
  html += "    const msgClass = isSystem ? 'msg-system' : 'msg-user';\n";
  html += "    history.innerHTML += '<div class=\"' + msgClass + '\">' + evt.data + '</div>';\n";
  html += "    history.scrollTop = history.scrollHeight; // Auto scroll\n";
  html += "  };\n";
  
  html += "  function sendChat() {\n";
  html += "    const text = input.value.trim();\n";
  html += "    if (text.length > 0) {\n";
  html += "      ws.send(text);\n";
  html += "      input.value = ''; // Clear input\n";
  html += "    }\n";
  html += "  }\n";
  
  html += "  function handleKeyPress(e) {\n";
  html += "    if (e.key === 'Enter') sendChat();\n";
  html += "  }\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\nESP32 Chat Terminal Server Starting...");
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
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Open two separate web browser windows and navigate to the printed IP address.
4. Type messages in either window. Verify that they are relayed to both windows instantly.

## Expected Output
Serial Monitor:
```
ESP32 Chat Terminal Server Starting...
....
WiFi Connected successfully.
HTTP Server active. Connect at: http://10.10.0.3

[Chat Alert] User #0 joined the chat.
[Chat Alert] User #1 joined the chat.
User #0: Hello everyone!
User #1: Hey #0, this is running over WebSockets!
```

## Expected Canvas Behavior
* Chatting in either window logs conversation threads immediately on the Serial Monitor.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `webSocket.broadcastTXT(...)` | Relays incoming messages to all connected subscribers. |
| `ws.send(text)` | Client-side command to send text messages over the active socket connection. |
| `evt.data.startsWith('SYSTEM:')` | Formats connection status logs differently from user chat messages. |

## Hardware & Safety Concept: Network Chat Security and Broadcast Routing
Persistently connected messaging systems relay messages directly to all subscribers. To prevent memory exhaustion:
1. **Message Limits**: Restrict the maximum message length (e.g. under 256 characters) to avoid buffer overflow on the ESP32 server.
2. **Rate Limiting**: Add a minor delay or throttle check to prevent a single client from flooding the chat room with messages.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the last message received.
2. **Buzzer notification tone**: Sound a quick beep on a buzzer (GPIO 15) when a new user joins.
3. **Private messaging**: Implement commands (like `/msg #0 hello`) to send private messages to specific clients.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Messages are not received | connection failed | Open the browser's developer console (F12) to inspect JavaScript console errors |
| The server crashes on large messages | Stack overflow | Restrict message buffer allocations in the WebSocket handler |
| Chat history overflows card | scroll styling missing | Ensure the container uses `overflow-y: auto` styling |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [71 - ESP32 WebSocket Server initialization](71-esp32-websocket-server-initialization.md)
- [72 - ESP32 WebSocket LED Toggle (low latency feedback)](72-esp32-websocket-led-toggle-low-latency-feedback.md)
- [74 - ESP32 WebSocket Dual-Axis Joystick Controller](74-esp32-websocket-dual-axis-joystick-controller.md)
