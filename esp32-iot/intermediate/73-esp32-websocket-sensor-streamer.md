# 73 - WebSocket Sensor Streamer (Real-time ADC graphing)

Build a high-speed telemetry station on the ESP32 that establishes a full-duplex WebSocket channel, samples an analog potentiometer sensor on GPIO 34, and streams raw readings to connected web clients at 20 Hz to plot real-time scrolling charts.

## Goal
Learn how to implement high-speed WebSocket streaming loops, manage client broadcasting, parse data streams, and build real-time visual charts on web canvases.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It hosts a webpage on port 80 and a WebSocket server on port 81. A potentiometer is on GPIO 34. Navigating to the page opens a WebSocket channel. The ESP32 streams the raw sensor value every 50 ms (20 Hz). JavaScript on the webpage receives the values and plots them onto a scrolling line graph in real time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Wiper | GPIO34 | Yellow | Analog sensor signal |
| Shared power | VCC / GND | 3V3 / GND | Red / Black | Sensor power rails |

> **Wiring tip:** Connect the potentiometer wiper to GPIO 34.

## Code
```cpp
// WebSocket Sensor Streamer (High-speed real-time ADC grapher)
#include <WiFi.h>
#include <WebServer.h>
#include <WebSocketsServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int POT_PIN = 34;

WebServer server(80);
WebSocketsServer webSocket = WebSocketsServer(81);

unsigned long lastStreamTime = 0;
const unsigned long STREAM_INTERVAL_MS = 50; // Stream at 20 Hz (every 50 ms)

// WebSocket Event Callback Handler
void webSocketEvent(uint8_t num, WStype_t type, uint8_t * payload, size_t length) {
  switch (type) {
    case WStype_DISCONNECTED:
      Serial.printf("[WebSocket] Subscriber #%u left.\n", num);
      break;
    case WStype_CONNECTED:
      Serial.printf("[WebSocket] Subscriber #%u joined.\n", num);
      break;
    case WStype_TEXT:
      // No input commands needed for this streamer node
      break;
  }
}

// Root URL Handler ("/")
// Serves the HTML dashboard containing CSS, the Canvas element, and client-side graphing script
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>WebSocket Grapher</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 550px; width: 100%; text-align: center; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; margin-bottom: 10px; }\n";
  html += "  .value { font-size: 32px; font-weight: bold; color: #10b981; margin-bottom: 20px; font-family: monospace; }\n";
  html += "  canvas { background-color: #0f172a; border-radius: 8px; border: 1px solid #334155; width: 100%; height: 200px; }\n";
  html += "  .footer { text-align: center; margin-top: 20px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Real-Time Sensor Plotter</h1>\n";
  html += "  <div id=\"adcVal\" class=\"value\">ADC: 0</div>\n";
  
  // Graph Canvas
  html += "  <canvas id=\"chartCanvas\" width=\"500\" height=\"200\"></canvas>\n";
  html += "  <p class=\"footer\">WebSocket stream rate: 20 Hz (50 ms packets)</p>\n";
  html += "</div>\n";
  
  // Client-side graphing script
  html += "<script>\n";
  html += "  const canvas = document.getElementById('chartCanvas');\n";
  html += "  const ctx = canvas.getContext('2d');\n";
  html += "  const dataBuffer = new Array(100).fill(0); // Scroll buffer for 100 points\n";
  
  // Function to draw line chart on Canvas
  html += "  function drawChart() {\n";
  html += "    ctx.clearRect(0, 0, canvas.width, canvas.height);\n";
  html += "    ctx.strokeStyle = '#38bdf8';\n";
  html += "    ctx.lineWidth = 3;\n";
  html += "    ctx.beginPath();\n";
  
  html += "    for (let i = 0; i < dataBuffer.length; i++) {\n";
  html += "      const x = (i / (dataBuffer.length - 1)) * canvas.width;\n";
  // Map value range (0-4095) to canvas height (height to 0 because y axis is inverted in canvas)
  html += "      const y = canvas.height - (dataBuffer[i] / 4095) * canvas.height;\n";
  html += "      if (i === 0) ctx.moveTo(x, y); else ctx.lineTo(x, y);\n";
  html += "    }\n";
  html += "    ctx.stroke();\n";
  html += "  }\n";
  
  // Establish WebSocket connection
  html += "  const ws = new WebSocket('ws://' + window.location.hostname + ':81/');\n";
  
  html += "  ws.onmessage = function(evt) {\n";
  html += "    const rawVal = parseInt(evt.data);\n";
  html += "    document.getElementById('adcVal').innerText = 'ADC: ' + rawVal;\n";
  
  // Shift scroll buffer
  html += "    dataBuffer.shift();\n";
  html += "    dataBuffer.push(rawVal);\n";
  
  // Re-draw chart frame
  html += "    drawChart();\n";
  html += "  };\n";
  
  html += "  drawChart(); // Initial clean canvas\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(POT_PIN, INPUT);
  
  Serial.println("\nESP32 WebSocket Streamer Starting...");
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
  
  // Broadcast sensor value periodically (every 50 ms = 20 Hz)
  unsigned long now = millis();
  if (now - lastStreamTime >= STREAM_INTERVAL_MS) {
    if (webSocket.connectedClients() > 0) {
      int potVal = analogRead(POT_PIN);
      
      // Broadcast raw value as text
      webSocket.broadcastTXT(String(potVal).c_str());
    }
    lastStreamTime = now;
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Potentiometer** onto the canvas.
2. Wire Potentiometer to **GPIO34**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Slide the potentiometer widget. Watch the graph line trace the motion in real time.

## Expected Output
Serial Monitor:
```
ESP32 WebSocket Streamer Starting...
....
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[WebSocket] Subscriber #0 joined.
```

Browser Screen:
* A scrolling light-blue line graph moving from right to left, plotting the active potentiometer position.

## Expected Canvas Behavior
* Adjusting the potentiometer updates the line height on the webpage chart immediately.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `webSocket.connectedClients() > 0` | Checks if there are active subscribers before performing ADC reads, reducing CPU overhead. |
| `webSocket.broadcastTXT(...)` | Transmits the reading to all connected web page clients. |
| `dataBuffer.shift() / push()` | Shifts the scrolling data buffer to plot the newest points. |
| `ctx.lineTo(x, y)` | Draws a line segment between buffer coordinates on the canvas. |

## Hardware & Safety Concept: Network Throughput and Refresh Rates
High-speed data plotting generates significant network traffic. When streaming data:
1. **Header Overhead**: Traditional HTTP polling (GET requests) sends headers (several hundred bytes) for every sensor reading. At 20 Hz, this would overwhelm the network.
2. **WebSockets efficiency**: WebSockets package data into frames with only a 2-byte header, enabling high-frequency streams with minimal overhead.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a checklist showing which clients are active.
2. **Frequency Selector**: Add a slider on the webpage to adjust the stream frequency (e.g. 5 Hz to 50 Hz).
3. **Save to File**: Add a button on the webpage to download the chart data buffer as a CSV file.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The graph line is static or flat | WebSocket disconnected | Confirm that the WebSocket server is running on port 81 and active |
| The graph updates slowly or lags | blocking delays in loop | Verify that the loop function has no blocking delay calls |
| Values are out of range | canvas coordinate mapping error | Check the canvas scale division math in the script |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [26 - ESP32 TCP Telemetry Broadcaster](../beginner/26-esp32-tcp-telemetry-broadcaster.md)
- [61 - ESP32 Web Server AJAX Telemetry Auto-refresh](61-esp32-web-server-ajax-telemetry-auto-refresh.md)
- [72 - ESP32 WebSocket LED Toggle (low latency feedback)](72-esp32-websocket-led-toggle-low-latency-feedback.md)
