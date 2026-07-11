# 74 - WebSocket Dual-Axis Joystick Controller

Build a web-based remote control station on the ESP32 that establishes a full-duplex WebSocket channel, hosts a webpage containing a virtual 2D joystick pad, parses incoming coordinate packets (`X:90,Y:90`), and drives two servo motors connected to GPIO 13 and 12.

## Goal
Learn how to parse coordinate string payloads, control multiple servo motors using `ESP32Servo`, handle touch and mouse events in JavaScript, and manage high-speed WebSocket command streams.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It hosts a webpage on port 80 and a WebSocket server on port 81. Two servo motors are connected to GPIO 13 (X-axis) and GPIO 12 (Y-axis). Navigating to the page displays a virtual joystick pad. Dragging the joystick handle sends coordinate updates, positioning the two servos instantly in real time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Servo Motor (X-Axis) | `servo` | Yes | Yes |
| Servo Motor (Y-Axis) | `servo` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Servo X | PWM (Signal) | GPIO13 | Orange | Horizontal axis servo |
| Servo Y | PWM (Signal) | GPIO12 | Yellow | Vertical axis servo |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power both servos from the 5V Vin rail.

## Code
```cpp
// WebSocket Dual-Axis Joystick Controller (Remote Gimbal Control)
#include <WiFi.h>
#include <WebServer.h>
#include <WebSocketsServer.h>
#include <ESP32Servo.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Configure Servo Pins
const int SERVO_X_PIN = 13;
const int SERVO_Y_PIN = 12;

Servo servoX;
Servo servoY;

WebServer server(80);
WebSocketsServer webSocket = WebSocketsServer(81);

// WebSocket Event Callback Handler
void webSocketEvent(uint8_t num, WStype_t type, uint8_t * payload, size_t length) {
  switch (type) {
    case WStype_DISCONNECTED:
      Serial.printf("[WebSocket] Controller #%u disconnected.\n", num);
      break;
      
    case WStype_CONNECTED:
      Serial.printf("[WebSocket] Controller #%u connected.\n", num);
      break;
      
    case WStype_TEXT: {
      String msg = String((char*)payload);
      
      // Expected format: "X:90,Y:90"
      int xIdx = msg.indexOf("X:");
      int yIdx = msg.indexOf(",Y:");
      
      if (xIdx >= 0 && yIdx >= 0) {
        // Extract coordinate strings and convert to integers
        int xVal = msg.substring(xIdx + 2, yIdx).toInt();
        int yVal = msg.substring(yIdx + 3).toInt();
        
        // Safety range check
        if (xVal >= 0 && xVal <= 180 && yVal >= 0 && yVal <= 180) {
          servoX.write(xVal);
          servoY.write(yVal);
          
          static unsigned long lastLog = 0;
          if (millis() - lastLog >= 200) { // Limit log prints to 5 Hz
            Serial.printf("[Server] Gimbal coordinates -> X: %d | Y: %d\n", xVal, yVal);
            lastLog = millis();
          }
        }
      }
      break;
    }
  }
}

// Root URL Handler ("/")
// Serves the HTML page containing the virtual joystick touchpad
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0, user-scalable=no\">\n";
  html += "<title>Smart Gimbal Control</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; touch-action: none; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 350px; width: 100%; text-align: center; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; margin-bottom: 25px; }\n";
  
  // Joystick Touchpad Design
  html += "  .joystick-pad { width: 200px; height: 200px; background-color: #0f172a; border-radius: 50%; border: 3px solid #334155; margin: 0 auto 20px auto; position: relative; overflow: hidden; }\n";
  html += "  .joystick-knob { width: 60px; height: 60px; background-color: #38bdf8; border-radius: 50%; position: absolute; left: 70px; top: 70px; cursor: pointer; box-shadow: 0 4px 10px rgba(0,0,0,0.3); transition: background-color 0.2s; }\n";
  html += "  .joystick-knob:active { background-color: #0ea5e9; }\n";
  html += "  .status { font-family: monospace; font-size: 14px; color: #94a3b8; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Gimbal Controller</h1>\n";
  
  // 2D Touchpad
  html += "  <div class=\"joystick-pad\" id=\"pad\">\n";
  html += "    <div class=\"joystick-knob\" id=\"knob\"></div>\n";
  html += "  </div>\n";
  
  html += "  <div id=\"coords\" class=\"status\">Coordinates - X: 90, Y: 90</div>\n";
  html += "</div>\n";
  
  // Client-side Touch and Mouse handler script
  html += "<script>\n";
  html += "  const pad = document.getElementById('pad');\n";
  html += "  const knob = document.getElementById('knob');\n";
  html += "  const coords = document.getElementById('coords');\n";
  
  html += "  const ws = new WebSocket('ws://' + window.location.hostname + ':81/');\n";
  
  html += "  let active = false;\n";
  html += "  const padRadius = 100;\n";
  html += "  const knobRadius = 30;\n";
  
  html += "  function handleMove(clientX, clientY) {\n";
  html += "    if (!active) return;\n";
  html += "    const rect = pad.getBoundingClientRect();\n";
  html += "    let x = clientX - rect.left - padRadius;\n";
  html += "    let y = clientY - rect.top - padRadius;\n";
  
  // Restrict knob inside pad boundary
  html += "    const dist = Math.sqrt(x*x + y*y);\n";
  html += "    const maxDist = padRadius - knobRadius;\n";
  html += "    if (dist > maxDist) {\n";
  html += "      x = (x / dist) * maxDist;\n";
  html += "      y = (y / dist) * maxDist;\n";
  html += "    }\n";
  
  // Position knob visual element
  html += "    knob.style.left = (x + padRadius - knobRadius) + 'px';\n";
  html += "    knob.style.top = (y + padRadius - knobRadius) + 'px';\n";
  
  // Map coordinate offsets (-70 to 70) to servo range (0 to 180)
  html += "    const servoX = Math.round(((x / maxDist) + 1) * 90);\n";
  html += "    const servoY = Math.round(((y / maxDist) + 1) * 90);\n";
  
  html += "    coords.innerText = 'Coordinates - X: ' + servoX + ', Y: ' + servoY;\n";
  
  // Send coordinates over WebSocket channel
  html += "    if (ws.readyState === WebSocket.OPEN) {\n";
  html += "      ws.send('X:' + servoX + ',Y:' + servoY);\n";
  html += "    }\n";
  html += "  }\n";
  
  html += "  function resetPosition() {\n";
  html += "    active = false;\n";
  // Center knob
  html += "    knob.style.left = (padRadius - knobRadius) + 'px';\n";
  html += "    knob.style.top = (padRadius - knobRadius) + 'px';\n";
  html += "    coords.innerText = 'Coordinates - X: 90, Y: 90';\n";
  html += "    if (ws.readyState === WebSocket.OPEN) ws.send('X:90,Y:90');\n";
  html += "  }\n";
  
  // Event listeners for Mouse and Touch
  html += "  knob.addEventListener('mousedown', () => active = true);\n";
  html += "  window.addEventListener('mouseup', resetPosition);\n";
  html += "  window.addEventListener('mousemove', (e) => handleMove(e.clientX, e.clientY));\n";
  
  html += "  knob.addEventListener('touchstart', () => active = true);\n";
  html += "  window.addEventListener('touchend', resetPosition);\n";
  html += "  window.addEventListener('touchmove', (e) => {\n";
  html += "    if (e.touches.length > 0) handleMove(e.touches[0].clientX, e.touches[0].clientY);\n";
  html += "  });\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Attach Servo motors
  servoX.attach(SERVO_X_PIN);
  servoY.attach(SERVO_Y_PIN);
  
  // Center servos on startup
  servoX.write(90);
  servoY.write(90);
  
  Serial.println("\nESP32 Gimbal Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
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
1. Drag **ESP32 DevKitC** and **2 Servos** onto the canvas.
2. Wire Servo X to **GPIO13** and Servo Y to **GPIO12**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Drag the virtual joystick knob. Watch the two servos rotate in sync.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[WebSocket] Controller #0 connected.
[Server] Gimbal coordinates -> X: 45 | Y: 110
[Server] Gimbal coordinates -> X: 20 | Y: 140
```

## Expected Canvas Behavior
* Dragging the virtual joystick updates the position of both servo widgets in real time.
* Releasing the joystick centers both servos (90 degrees).

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `servoX.write(xVal)` | Positions the horizontal axis servo. |
| `servoY.write(yVal)` | Positions the vertical axis servo. |
| `ws.send('X:' + ...)` | Transmits coordinate pairs as text packets over the open socket channel. |
| `resetPosition()` | Resets the joystick knob position and centers both servos when touch is released. |

## Hardware & Safety Concept: Actuator Speed Limits and Current Spikes
Moving multiple high-torque servos simultaneously draws significant current (up to 1 A spike). If the power supply is weak, this will drop the voltage and reset the ESP32. To protect the system, power the servos from the **5V Vin rail** or an external supply, and limit the rate of position updates to prevent excessive current draw.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a 2D crosshair representing the current coordinate position.
2. **Reverse direction toggle**: Add a toggle switch on the webpage to invert the servo direction.
3. **Buzzer limit alert**: Sound a quick beep on a buzzer (GPIO 15) when any servo reaches its physical limits (0 or 180 degrees).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servos do not move | power rails missing | Confirm that the servos are powered from the 5V Vin rail, not the 3.3V rail |
| Jittery movement | loop function blocked | Verify that the loop function has no blocking delay calls |
| Coordinates are swapped | wiring crossed | Verify that Servo X is wired to GPIO 13 and Servo Y to GPIO 12 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [56 - ESP32 Web Page Servo angle controller](56-esp32-web-page-servo-angle-controller-oled-display-feedback.md)
- [72 - ESP32 WebSocket LED Toggle (low latency feedback)](72-esp32-websocket-led-toggle-low-latency-feedback.md)
- [75 - ESP32 WebSocket Chat Terminal](75-esp32-websocket-chat-terminal-device-to-device-text-chat.md) (Next project)
