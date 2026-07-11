# 115 - Keypad Web Display HUD (Keypad + LCD + Web chat terminal)

Build a bidirectional chat console on the ESP32 that displays keys typed on a physical 4x4 matrix keypad on a 16x2 I2C LCD, transmits text messages to connected web browsers over a full-duplex WebSocket channel on port 81, and displays incoming messages from the webpage back on the LCD.

## Goal
Learn how to scan matrix keypads, print text dynamically to I2C LCDs, establish low-latency WebSocket communication, and manage multi-client broadcasting.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A 4x4 matrix keypad is connected to GPIO pins, and a 16x2 LCD on I2C (GPIO 21/22). Typing characters on the keypad displays them on the LCD. Pressing `#` sends the message over WebSockets to all connected web clients. Simultaneously, typing a message on the webpage sends it to the ESP32, displaying it on the second row of the LCD.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| 4x4 Keypad | Rows 1 - 4 | GPIO12, 14, 27, 26 | Yellow | Row scan lines |
| 4x4 Keypad | Cols 1 - 4 | GPIO25, 33, 32, 35 | Green | Column scan lines |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the keypad and LCD from the 5V Vin rail.

## Code
```cpp
// Keypad Web Display HUD (Keypad entry relay + I2C LCD + WebSockets Messenger)
#include <WiFi.h>
#include <WebServer.h>
#include <WebSocketsServer.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Keypad.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

const char* ssid = "Wokwi-GUEST";
const char* password = "";

WebServer server(80);
WebSocketsServer webSocket = WebSocketsServer(81);

// Keypad configuration
const byte ROWS = 4;
const byte COLS = 4;
char keys[ROWS][COLS] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};
byte rowPins[ROWS] = {12, 14, 27, 26}; // Row pinouts
byte colPins[COLS] = {25, 33, 32, 35}; // Column pinouts

Keypad customKeypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);

// Local typing buffer
String localMessageBuffer = "";
String remoteMessage = "No new messages";

void updateLCDDisplay() {
  lcd.clear();
  
  // Row 0: Local typing buffer
  lcd.setCursor(0, 0);
  lcd.print("Local: ");
  lcd.print(localMessageBuffer);
  
  // Row 1: Last received message from web
  lcd.setCursor(0, 1);
  lcd.print("Web:   ");
  lcd.print(remoteMessage);
}

// WebSocket Event Callback Handler
void webSocketEvent(uint8_t num, WStype_t type, uint8_t * payload, size_t length) {
  switch (type) {
    case WStype_DISCONNECTED:
      Serial.printf("[WebSocket] Client #%u disconnected.\n", num);
      break;
      
    case WStype_CONNECTED:
      Serial.printf("[WebSocket] Client #%u connected.\n", num);
      webSocket.sendTXT(num, "SYSTEM: Connected to ESP32 HUD Terminal.");
      break;
      
    case WStype_TEXT: {
      String msg = String((char*)payload);
      msg.trim();
      
      if (msg.length() > 0) {
        // Update remote message cache
        remoteMessage = msg;
        
        // Truncate message to fit on 16-character LCD
        if (remoteMessage.length() > 10) {
          remoteMessage = remoteMessage.substring(0, 10);
        }
        
        Serial.printf("[WebSocket] Message from client #%u: %s\n", num, msg.c_str());
        updateLCDDisplay();
        
        // Broadcast new message to all other connected clients
        String relayMsg = "User #" + String(num) + ": " + msg;
        webSocket.broadcastTXT(relayMsg.c_str());
      }
      break;
    }
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>HUD Communication Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; display: flex; flex-direction: column; height: 80vh; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; text-align: center; }\n";
  html += "  #chatHistory { flex-grow: 1; background-color: #0f172a; border-radius: 8px; padding: 15px; overflow-y: auto; text-align: left; font-size: 14px; border: 1px solid #334155; margin-bottom: 20px; font-family: monospace; color: #10b981; }\n";
  html += "  .msg-system { color: #64748b; font-style: italic; margin-bottom: 10px; }\n";
  html += "  .msg-user { color: #f1f5f9; margin-bottom: 10px; }\n";
  html += "  .input-container { display: flex; gap: 10px; }\n";
  html += "  #messageInput { flex-grow: 1; padding: 12px; border-radius: 6px; border: 1px solid #334155; background-color: #0f172a; color: white; font-size: 14px; }\n";
  html += "  #messageInput:focus { outline: none; border-color: #38bdf8; }\n";
  html += "  .btn { padding: 12px 24px; font-size: 14px; font-weight: bold; color: white; background-color: #38bdf8; border: none; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; }\n";
  html += "  .btn:hover { background-color: #0ea5e9; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>HUD Chat Console</h1>\n";
  html += "  <div id=\"chatHistory\"></div>\n";
  
  html += "  <div class=\"input-container\">\n";
  html += "    <input type=\"text\" id=\"messageInput\" placeholder=\"Type message to LCD...\" onkeypress=\"handleKeyPress(event)\">\n";
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
  html += "    history.scrollTop = history.scrollHeight;\n";
  html += "  };\n";
  
  html += "  function sendChat() {\n";
  html += "    const text = input.value.trim();\n";
  html += "    if (text.length > 0) {\n";
  html += "      ws.send(text);\n";
  html += "      input.value = '';\n";
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
  
  // Initialize LCD screen
  lcd.init();
  lcd.backlight();
  
  lcd.setCursor(0, 0);
  lcd.print("HUD Setup");
  lcd.setCursor(0, 1);
  lcd.print("Connecting WiFi");
  
  Serial.println("\nESP32 Chat HUD Station Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Start server
  server.on("/", handleRoot);
  server.begin();
  
  // Start WebSocket Server
  webSocket.begin();
  webSocket.onEvent(webSocketEvent);
  
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  updateLCDDisplay();
}

void loop() {
  server.handleClient();
  webSocket.loop();
  
  // 1. Scan keypad for button clicks
  char key = customKeypad.getKey();
  
  if (key) {
    Serial.printf("[Keypad] Pressed: %c\n", key);
    
    // Clear buffer if '*' is pressed
    if (key == '*') {
      localMessageBuffer = "";
      updateLCDDisplay();
    } 
    // Transmit buffer to web clients if '#' is pressed
    else if (key == '#') {
      if (localMessageBuffer.length() > 0) {
        String payload = "Keypad: " + localMessageBuffer;
        
        // Broadcast over WebSocket channel
        webSocket.broadcastTXT(payload.c_str());
        
        // Reset local buffer
        localMessageBuffer = "";
        updateLCDDisplay();
      }
    } 
    // Append characters to buffer (limit length to 10 characters to fit LCD)
    else {
      if (localMessageBuffer.length() < 10) {
        localMessageBuffer += key;
        updateLCDDisplay();
      }
    }
  }
  
  delay(1);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Keypad**, and **16x2 I2C LCD** onto the canvas.
2. Wire Keypad rows/cols to **GPIO pins** matching the pinout tables, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Click keys `1`, `2`, `3`, and `#` on the keypad widget. Verify that "Keypad: 123" displays on the webpage console.
6. Type a message in the text box on the webpage and click "Send". Verify that the message displays on the second row of the LCD.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Keypad] Pressed: 1
[Keypad] Pressed: 2
[Keypad] Pressed: 3
[Keypad] Pressed: #
[WebSocket] Message from client #0: Hello LCD
```

LCD Display:
```
Local:
Web:   Hello LCD
```

## Expected Canvas Behavior
* Pressing buttons on the keypad widget displays them on the first row of the LCD widget.
* Typing a message on the webpage updates the second row of the LCD widget instantly.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `customKeypad.getKey()` | Scans the row and column matrix pins for closed contacts. |
| `key == '#'` | Standard confirmation key to transmit the typed buffer. |
| `webSocket.broadcastTXT(...)` | Transmits the message to all connected web clients. |
| `remoteMessage = msg` | Caches the message received from the web page client. |

## Hardware & Safety Concept: Bidirectional Buffer Constraints and LCD Formatting
* **LCD Constraints**: 16x2 character displays can only show 16 characters per line. If a message is too long, it overflows and cuts off. Truncate incoming messages to 10 characters (e.g. `remoteMessage.substring(0, 10)`) to leave room for the prefix (`Web: `).
* **Buffer Overflow Prevention**: Matrix keypads do not have backspace keys. Implement a clear key (like `*`) to let users reset the buffer and start over if they make a mistake.

## Try This! (Challenges)
1. **OLED Status Display**: Upgrade the screen to an SSD1306 OLED (Project 60) and display a scrolling chat history.
2. **Alert chime buzzer**: Sound a brief click tone on a buzzer (GPIO 15) when a key is pressed.
3. **SPIFFS integration**: Log chat conversations to a file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Keypad clicks are ignored | Pin mapping wrong | Confirm that row and column pins match the pin configurations in the code |
| LCD does not print text | I2C address mismatch | Verify the I2C address in `LiquidCrystal_I2C` matches your LCD module (usually `0x27`) |
| Webpage console shows no output | connection failed | Open the browser's developer console (F12) to inspect JavaScript console errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [37 - ESP32 LCD Network Clock](../beginner/37-esp32-lcd-network-clock.md)
- [75 - ESP32 WebSocket Chat Terminal](75-esp32-websocket-chat-terminal-device-to-device-text-chat.md)
- [114 - ESP32 Keypad 4x4 Password Lock IoT](114-esp32-keypad-4x4-password-lock-iot-keypad-servo-web-access-logs.md)
