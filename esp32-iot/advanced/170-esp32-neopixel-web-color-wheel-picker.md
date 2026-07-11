# 170 - NeoPixel Web Color Wheel picker

Build an interactive, low-latency lighting controller on the ESP32 that drives a 16-pixel WS2812B NeoPixel ring on GPIO 12, hosts a web server on port 80, and runs a WebSocket server on port 81 to parse color wheel RGB and brightness coordinates from an SVG color wheel.

## Goal
Learn how to configure WebSocket servers for low-latency bidirectional communications, parse string buffers, drive WS2812B addressable LED strings, and build responsive WebGL/SVG color wheels.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network, hosting an HTTP server and a WebSocket endpoint. A 16-LED NeoPixel ring is on GPIO 12. Navigating to the ESP32's IP address displays a webpage with an interactive SVG color wheel and a brightness slider. Dragging your finger or cursor over the wheel sends selected RGB codes to the ESP32 over WebSockets (latency < 10 ms). The NeoPixels update color and brightness instantly.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| WS2812B NeoPixel Ring (16 LEDs) | `neopixel` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, verify color updates on the NeoPixel ring widget by dragging the mouse over the webpage color wheel.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NeoPixel Ring | DI (Data Input) | GPIO12 | Yellow | Serial data line |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the NeoPixel ring from the 5V Vin pin and place a 470 Ω resistor in series with the DI line to protect the first LED.

## Code
```cpp
// NeoPixel Web Color Wheel picker (WebSocket low-latency server + WS2812B addressable LED ring)
#include <WiFi.h>
#include <WebServer.h>
#include <WebSocketsServer.h>
#include <Adafruit_NeoPixel.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int NEOPIXEL_PIN = 12;
const int NUM_PIXELS = 16;

Adafruit_NeoPixel pixels(NUM_PIXELS, NEOPIXEL_PIN, NEO_GRB + NEO_KHZ800);

WebServer server(80);
WebSocketsServer webSocket = WebSocketsServer(81);

// Global color states
int currentRed = 0;
int currentGreen = 180;
int currentBlue = 240;
int currentBrightness = 100; // 0 to 255

// Update all NeoPixels with active color and brightness scaling
void updateNeoPixels() {
  for (int i = 0; i < NUM_PIXELS; i++) {
    // Scale color values by brightness
    int r = (currentRed * currentBrightness) / 255;
    int g = (currentGreen * currentBrightness) / 255;
    int b = (currentBlue * currentBrightness) / 255;
    pixels.setPixelColor(i, pixels.Color(r, g, b));
  }
  pixels.show();
}

// WebSocket Event Handler (Receives real-time color coordinates)
void onWebSocketEvent(uint8_t num, WStrustType_t type, uint8_t * payload, size_t length) {
  if (type == WStrustType_TEXT) {
    String textMsg = String((char*)payload);
    
    // Packet structure: "R:255,G:0,B:0,BR:128"
    if (textMsg.startsWith("R:")) {
      int rIndex = textMsg.indexOf("R:");
      int gIndex = textMsg.indexOf(",G:");
      int bIndex = textMsg.indexOf(",B:");
      int brIndex = textMsg.indexOf(",BR:");
      
      if (gIndex != -1 && bIndex != -1 && brIndex != -1) {
        currentRed = textMsg.substring(rIndex + 2, gIndex).toInt();
        currentGreen = textMsg.substring(gIndex + 3, bIndex).toInt();
        currentBlue = textMsg.substring(bIndex + 3, brIndex).toInt();
        currentBrightness = textMsg.substring(brIndex + 4).toInt();
        
        // Render colors instantly
        updateNeoPixels();
      }
    }
  }
}

// Serve root webpage containing interactive SVG color wheel
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>NeoPixel Color Wheel</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .wheel-container { position: relative; width: 260px; height: 260px; margin: 25px auto; }\n";
  html += "  #colorCanvas { border-radius: 50%; width: 100%; height: 100%; cursor: crosshair; box-shadow: inset 0 0 20px rgba(0,0,0,0.6); }\n";
  html += "  .tuning-box { border-top: 2px solid #334155; padding-top: 20px; margin-top: 20px; text-align: left; }\n";
  html += "  .slider-row { display: flex; align-items: center; gap: 15px; margin-bottom: 12px; }\n";
  html += "  label { font-weight: bold; color: #94a3b8; width: 100px; font-size: 13px; }\n";
  html += "  input[type=range] { flex-grow: 1; height: 8px; border-radius: 4px; background: #334155; outline: none; -webkit-appearance: none; }\n";
  html += "  input[type=range]::-webkit-slider-thumb { -webkit-appearance: none; width: 18px; height: 18px; border-radius: 50%; background: #38bdf8; cursor: pointer; }\n";
  html += "  .value-lbl { font-family: monospace; font-size: 15px; font-weight: bold; width: 45px; text-align: right; color: #38bdf8; }\n";
  html += "  .footer { text-align: center; margin-top: 25px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>NeoPixel Color Wheel</h1>\n";
  
  // Interactive Canvas Color Wheel
  html += "  <div class=\"wheel-container\">\n";
  html += "    <canvas id=\"colorCanvas\" width=\"300\" height=\"300\"></canvas>\n";
  html += "  </div>\n";
  
  html += "  <div class=\"tuning-box\">\n";
  html += "    <div class=\"slider-row\"><label>Brightness</label><input type=\"range\" id=\"brightSldr\" min=\"0\" max=\"255\" step=\"5\" value=\"" + String(currentBrightness) + "\" oninput=\"sendColor()\"><span id=\"brightVal\" class=\"value-lbl\">" + String(currentBrightness) + "</span></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">WebSocket server: Port 81 | Latency: &lt;10 ms</p>\n";
  html += "</div>\n";
  
  // Canvas color wheel draw & WebSocket communication script
  html += "<script>\n";
  html += "  const canvas = document.getElementById('colorCanvas');\n";
  html += "  const ctx = canvas.getContext('2d');\n";
  html += "  const cx = canvas.width / 2;\n";
  html += "  const cy = canvas.height / 2;\n";
  html += "  const r = cx - 5;\n";
  
  // Draw circular color wheel gradient
  html += "  for (let angle = 0; angle < 360; angle += 0.5) {\n";
  html += "    const startAngle = (angle - 0.5) * Math.PI / 180;\n";
  html += "    const endAngle = (angle + 0.5) * Math.PI / 180;\n";
  html += "    ctx.beginPath();\n";
  html += "    ctx.moveTo(cx, cy);\n";
  html += "    ctx.arc(cx, cy, r, startAngle, endAngle);\n";
  html += "    ctx.closePath();\n";
  
  // Map angle to HSL hue parameter
  html += "    const grad = ctx.createRadialGradient(cx, cy, 0, cx, cy, r);\n";
  html += "    grad.addColorStop(0, '#ffffff');\n";
  html += "    grad.addColorStop(1, 'hsl(' + angle + ', 100%, 50%)');\n";
  html += "    ctx.fillStyle = grad;\n";
  html += "    ctx.fill();\n";
  html += "  }\n";
  
  // WebSocket connection initialization
  html += "  let ws;\n";
  html += "  let selectedColor = { r: 0, g: 180, b: 240 };\n";
  
  html += "  function initWebSocket() {\n";
  html += "    ws = new WebSocket('ws://' + window.location.hostname + ':81/');\n";
  html += "    ws.onclose = function() { setTimeout(initWebSocket, 2000); };\n";
  html += "  }\n";
  
  html += "  function sendColor() {\n";
  html += "    const brightness = document.getElementById('brightSldr').value;\n";
  html += "    document.getElementById('brightVal').innerText = brightness;\n";
  
  html += "    if (ws && ws.readyState === WebSocket.OPEN) {\n";
  html += "      const msg = 'R:' + selectedColor.r + ',G:' + selectedColor.g + ',B:' + selectedColor.b + ',BR:' + brightness;\n";
  html += "      ws.send(msg);\n";
  html += "    }\n";
  html += "  }\n";
  
  // Capture clicks and drags on color wheel canvas
  html += "  function pickColor(e) {\n";
  html += "    const rect = canvas.getBoundingClientRect();\n";
  html += "    const scaleX = canvas.width / rect.width;\n";
  html += "    const scaleY = canvas.height / rect.height;\n";
  html += "    const x = (e.clientX - rect.left) * scaleX;\n";
  html += "    const y = (e.clientY - rect.top) * scaleY;\n";
  
  // Read pixel RGB values from canvas context
  html += "    const imgData = ctx.getImageData(x, y, 1, 1).data;\n";
  html += "    if (imgData[3] > 0) { // Check alpha channel to ignore clicks outside circle\n";
  html += "      selectedColor.r = imgData[0];\n";
  html += "      selectedColor.g = imgData[1];\n";
  html += "      selectedColor.b = imgData[2];\n";
  html += "      sendColor();\n";
  html += "    }\n";
  html += "  }\n";
  
  // Event listeners for smooth drag gestures
  html += "  let isDragging = false;\n";
  html += "  canvas.addEventListener('mousedown', (e) => { isDragging = true; pickColor(e); });\n";
  html += "  canvas.addEventListener('mousemove', (e) => { if (isDragging) pickColor(e); });\n";
  html += "  canvas.addEventListener('mouseup', () => { isDragging = false; });\n";
  html += "  canvas.addEventListener('touchstart', (e) => { isDragging = true; pickColor(e.touches[0]); });\n";
  html += "  canvas.addEventListener('touchmove', (e) => { if (isDragging) pickColor(e.touches[0]); });\n";
  html += "  canvas.addEventListener('touchend', () => { isDragging = false; });\n";
  
  html += "  window.onload = function() {\n";
  html += "    initWebSocket();\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Initialize WS2812B Strip
  pixels.begin();
  updateNeoPixels();
  
  Serial.println("\nESP32 NeoPixel Web Color starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Start Web Server on port 80
  server.on("/", handleRoot);
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  // Start WebSocket Server on port 81
  webSocket.begin();
  webSocket.onEvent(onWebSocketEvent);
  Serial.println("WebSocket Server active on port 81.");
}

void loop() {
  server.handleClient();
  webSocket.loop();
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and a **WS2812B NeoPixel Ring** onto the canvas.
2. Wire the DI pin of the NeoPixel Ring to **GPIO12**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. In normal state, click and drag your cursor over the colored wheel on the webpage.
6. Verify that the simulated NeoPixel LEDs update their color dynamically on the canvas, matching the color picked on the wheel.
7. Move the brightness slider. Verify that the LEDs dim and brighten.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
WebSocket Server active on port 81.
```

## Expected Canvas Behavior
* Clicking and dragging the cursor over the webpage color wheel changes the color of the simulated NeoPixel LED widgets instantly.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `webSocket.begin()` | Starts the WebSocket listening server on port 81. |
| `onWebSocketEvent(...)` | Event loop callback checking for incoming WebSocket text packets. |
| `textMsg.substring(...)` | Parses the raw coordinate string buffer into R, G, B integers. |
| `pixels.setPixelColor(i, ...)` | Sets individual WS2812B colors scaled by brightness. |

## Hardware & Safety Concept: Peak Current Draw and Snubber Resistors
* **Peak Current Draw**: Addressable WS2812B NeoPixel LEDs draw approximately 60 mA of current at full brightness (white). A 16-pixel ring operating at maximum brightness can draw: `16 * 60 mA = 960 mA` (nearly 1 Amp!). This exceeds the current capability of the ESP32's onboard voltage regulator. Power the NeoPixel ring from an external 5V power supply, and connect the grounds together.
* **Snubber Resistors**: The DI data line is high-frequency logic. Place a 300 to 500 Ω snubber resistor in series with the DI line close to the ring to prevent data reflection noise from corrupting commands. Solder a large electrolytic capacitor (e.g. 1000 µF) across the NeoPixel VCC and GND lines to smooth out rapid changes in current draw.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the current RGB value numerically.
2. **Animation Patterns**: Add a button on the webpage to switch the NeoPixels to a rainbow cycle animation when clicked.
3. **Sound level sync**: Add a microphone (Project 156) and flash the NeoPixel ring color in sync with ambient sound.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| NeoPixels light up in wrong colors | RGB / GRB order mismatch | Many NeoPixel rings use GRB pixel indexing instead of RGB. Change the NeoPixel definition parameter in code (e.g. `NEO_GRB` vs `NEO_RGB`) |
| WebSocket connection fails | Port 81 blocked | Ensure your browser is accessing the ESP32 IP directly and not blocking port 81 (check browser security settings) |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [60 - ESP32 OLED display driver](../intermediate/60-esp32-ssd1306-oled-display-driver.md)
- [121 - ESP32 WiFi Web Page Drive Controller](121-esp32-wifi-web-page-drive-controller.md)
- [169 - Multi-zone Temperature logger IoT](169-esp32-multi-zone-temperature-logger-iot-3x-ds18b20-cloud-databases.md)
- [171 - ESP32 WebSocket Game Controller](171-esp32-websocket-game-controller-hud.md) (Next project)
