# 133 - 2-Axis Robotic Arm Web Coordinate balancer

Build a web-controlled 2-axis robotic arm coordinator on the ESP32 that maps 2D coordinates (X, Y) from an interactive web-based touchpad to two servo motors (X-Axis on GPIO 12, Y-Axis on GPIO 13) representing Pan and Tilt joints.

## Goal
Learn how to map Cartesian coordinates to angular servo movements, design interactive HTML5 Canvas touchpads, stream coordinate updates using HTTP POST requests, and control multiple servos.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Two servos on GPIOs 12 and 13 control a 2-axis gimbal or arm mechanism. Navigating to the ESP32's IP address displays a webpage with a styled 2D touchpad box. Clicking or dragging inside this box tracks the cursor's (X, Y) coordinate and sends it to the ESP32. The ESP32 maps the coordinates to angles (0 to 180 degrees), rotating the servos to match the pointer position.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Servo Motors | `servo` | Yes (2 pieces) | Yes (2 pieces) |
| Shared power / Battery Pack | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, monitor the X and Y servo widgets to verify that they rotate in response to touchpad drags.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| X-Axis Servo | PWM (Signal) | GPIO12 | Yellow | Pan axis rotation |
| Y-Axis Servo | PWM (Signal) | GPIO13 | Orange | Tilt axis rotation |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the servos from a separate 5V battery pack and connect its ground to the ESP32 ground pin.

## Code
```cpp
// 2-Axis Robotic Arm Web Coordinate balancer (Touchpad XY Coordinate Map)
#include <WiFi.h>
#include <WebServer.h>
#include <ESP32Servo.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int X_SERVO_PIN = 12;
const int Y_SERVO_PIN = 13;

Servo xServo;
Servo yServo;

WebServer server(80);

// Normalized XY Coordinates (0 to 100%)
int xCoord = 50;
int yCoord = 50;

int xAngle = 90;
int yAngle = 90;

// HTTP API endpoint returning JSON data
void handleGetCoordinate() {
  String json = "{\"x\":" + String(xCoord) + 
                 ",\"y\":" + String(yCoord) + 
                 ",\"xAngle\":" + String(xAngle) +
                 ",\"yAngle\":" + String(yAngle) + "}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to update coordinates
void handlePostCoordinate() {
  if (server.hasArg("x") && server.hasArg("y")) {
    xCoord = server.arg("x").toInt();
    yCoord = server.arg("y").toInt();
    
    // Bounds check
    xCoord = constrain(xCoord, 0, 100);
    yCoord = constrain(yCoord, 0, 100);
    
    // Map normalized coordinates (0-100) to servo angles (0-180)
    xAngle = map(xCoord, 0, 100, 0, 180);
    yAngle = map(yCoord, 0, 100, 0, 180);
    
    // Drive servos
    xServo.write(xAngle);
    yServo.write(yAngle);
    
    Serial.printf("[Coord Map] Pos: (%d, %d) -> Angles: (%d, %d)\n", 
                  xCoord, yCoord, xAngle, yAngle);
                  
    server.send(200, "text/plain", "OK");
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>XY Coordinate Balancer</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  
  // Interactive touchpad styles
  html += "  .touchpad-container { width: 250px; height: 250px; background-color: #0f172a; border: 2px solid #334155; border-radius: 12px; margin: 25px auto; position: relative; cursor: crosshair; overflow: hidden; }\n";
  html += "  .crosshair-h { position: absolute; left: 0; width: 100%; height: 1px; background-color: #334155; top: 50%; }\n";
  html += "  .crosshair-v { position: absolute; top: 0; height: 100%; width: 1px; background-color: #334155; left: 50%; }\n";
  html += "  .pointer-dot { width: 16px; height: 16px; border-radius: 50%; background-color: #38bdf8; position: absolute; transform: translate(-50%, -50%); border: 2px solid white; box-shadow: 0 0 10px #38bdf8; top: 50%; left: 50%; pointer-events: none; }\n";
  
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 20px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 12px; border: 1px solid #334155; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 22px; font-weight: bold; margin-top: 5px; color: #10b981; font-family: monospace; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>XY Coordinate Balancer</h1>\n";
  
  // Touchpad Grid
  html += "  <div id=\"touchPad\" class=\"touchpad-container\" onmousedown=\"startDrag(event)\" onmousemove=\"drag(event)\" onmouseup=\"endDrag()\" onmouseleave=\"endDrag()\">\n";
  html += "    <div class=\"crosshair-h\"></div>\n";
  html += "    <div class=\"crosshair-v\"></div>\n";
  html += "    <div id=\"pointer\" class=\"pointer-dot\"></div>\n";
  html += "  </div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">X Coordinate (Pan)</div><div class=\"metric-val\" id=\"xVal\">50%</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Y Coordinate (Tilt)</div><div class=\"metric-val\" id=\"yVal\">50%</div></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Drag or click inside touchpad to rotate servos | Port 80</p>\n";
  html += "</div>\n";
  
  // Touchpad dragging Javascript logic
  html += "<script>\n";
  html += "  let isDragging = false;\n";
  html += "  const pad = document.getElementById('touchPad');\n";
  html += "  const ptr = document.getElementById('pointer');\n";
  
  html += "  function updatePointer(clientX, clientY) {\n";
  html += "    const rect = pad.getBoundingClientRect();\n";
  html += "    let x = clientX - rect.left;\n";
  html += "    let y = clientY - rect.top;\n";
  
  // Constrain to pad dimensions
  html += "    x = Math.max(0, Math.min(x, rect.width));\n";
  html += "    y = Math.max(0, Math.min(y, rect.height));\n";
  
  // Set pointer CSS positions
  html += "    ptr.style.left = x + 'px';\n";
  html += "    ptr.style.top = y + 'px';\n";
  
  // Calculate percentage (0-100)
  html += "    const pctX = Math.round((x / rect.width) * 100);\n";
  html += "    const pctY = Math.round((1 - (y / rect.height)) * 100); // Invert Y axis to match normal graph\n";
  
  html += "    document.getElementById('xVal').innerText = pctX + '%';\n";
  html += "    document.getElementById('yVal').innerText = pctY + '%';\n";
  
  // Send coordinates via POST
  html += "    sendCoordinates(pctX, pctY);\n";
  html += "  }\n";
  
  html += "  let lastSentTime = 0;\n";
  html += "  function sendCoordinates(x, y) {\n";
  html += "    const now = Date.now();\n";
  html += "    if (now - lastSentTime < 50) return; // Throttle to 20 requests per second\n";
  html += "    lastSentTime = now;\n";
  
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('x', x);\n";
  html += "    body.append('y', y);\n";
  html += "    fetch('/api/coordinate', { method: 'POST', body: body });\n";
  html += "  }\n";
  
  html += "  function startDrag(e) {\n";
  html += "    isDragging = true;\n";
  html += "    updatePointer(e.clientX, e.clientY);\n";
  html += "  }\n";
  
  html += "  function drag(e) {\n";
  html += "    if (isDragging) {\n";
  html += "      updatePointer(e.clientX, e.clientY);\n";
  html += "    }\n";
  html += "  }\n";
  
  html += "  function endDrag() {\n";
  html += "    isDragging = false;\n";
  html += "  }\n";
  
  // Prevent default scroll on touch devices
  html += "  pad.addEventListener('touchstart', function(e) {\n";
  html += "    isDragging = true;\n";
  html += "    updatePointer(e.touches[0].clientX, e.touches[0].clientY);\n";
  html += "    e.preventDefault();\n";
  html += "  });\n";
  html += "  pad.addEventListener('touchmove', function(e) {\n";
  html += "    if (isDragging) {\n";
  html += "      updatePointer(e.touches[0].clientX, e.touches[0].clientY);\n";
  html += "      e.preventDefault();\n";
  html += "    }\n";
  html += "  });\n";
  html += "  pad.addEventListener('touchend', function(e) { isDragging = false; });\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Attach Servos and move to center positions
  xServo.attach(X_SERVO_PIN);
  xServo.write(xAngle);
  
  yServo.attach(Y_SERVO_PIN);
  yServo.write(yAngle);
  
  Serial.println("\nESP32 Coordinate Balancer starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/coordinate", HTTP_POST, handlePostCoordinate);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **2 Servo Motors** (X-Axis, Y-Axis) onto the canvas.
2. Wire X-Axis Servo to **GPIO12** and Y-Axis Servo to **GPIO13**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Click and drag your mouse pointer inside the touchpad box. Verify that both simulated servo widgets on the canvas rotate in response to the cursor movement.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Coord Map] Pos: (20, 80) -> Angles: (36, 144)
[Coord Map] Pos: (21, 79) -> Angles: (37, 142)
```

## Expected Canvas Behavior
* Dragging inside the web touchpad rotates the X and Y servo widgets on the canvas in matching directions.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `xServo.attach(X_SERVO_PIN)` | Binds the pan axis servo to GPIO 12. |
| `map(xCoord, 0, 100, 0, 180)` | Maps the normalized X coordinate to X servo angle. |
| `constrain(xCoord, 0, 100)` | Ensures coordinates stay within 0-100% bounds. |
| `now - lastSentTime < 50` | Throttles web request rates to prevent overloading the ESP32. |

## Hardware & Safety Concept: Network Throttling and Mechanical Limits
* **Network Throttling**: Dragging a cursor on a webpage triggers mouse move events up to 100 times per second. Sending 100 HTTP POST requests per second over WiFi will flood the ESP32's network stack, causing connection timeouts. Implementing a **throttle delay** (e.g. `50 ms` minimum interval) limits requests to a manageable 20 per second.
* **Mechanical Limits**: Physical arms or camera gimbals often have structural limits (e.g. hitting frame brackets if tilted beyond 150 degrees). Set safety limits in the mapping function (e.g. `map(yCoord, 0, 100, 30, 150)`) to protect the hardware.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a dot mapping the (X, Y) coordinates.
2. **Alert Buzzer**: Add a buzzer (GPIO 15) to sound a chime when the pointer reaches the outer boundaries (0 or 100%).
3. **SPIFFS integration**: Save coordinates to SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servos jitter during dragging | Network lag or noise | Check the throttling timer in the Javascript code; ensure it is set to at least 50 ms |
| The pointer moves but servos do not rotate | Servo power missing | Verify that the servos are powered from a separate power supply |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [56 - ESP32 Web Page Servo angle controller](56-esp32-web-page-servo-angle-controller-oled-display-feedback.md)
- [132 - ESP32 Servo Robotic Arm Web Slider controls](132-esp32-servo-robotic-arm-web-slider-controls.md)
- [134 - ESP32 3-Axis Robotic Arm Web claw controller](134-esp32-3-axis-robotic-arm-web-claw-controller.md) (Next project)
