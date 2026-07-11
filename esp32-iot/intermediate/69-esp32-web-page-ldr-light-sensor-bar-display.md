# 69 - Web Page LDR light sensor bar display

Build a web-connected light meter on the ESP32 that hosts a control page, reads ambient light levels from an analog photoresistor (LDR) on GPIO 34, serves a JSON API endpoint, and uses client-side JavaScript (AJAX polling) to refresh a light intensity progress bar every second.

## Goal
Learn how to sample LDR photoresistors, map raw ADC inputs to percentages, implement client-side AJAX polling loops, and design visual progress bars.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a web server on port 80. An LDR is on GPIO 34. Navigating to the ESP32's IP address displays a webpage showing the ambient light level. JavaScript on the page polls `/api/light` every second, updating the percentage and the width of a progress bar immediately.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Photoresistor (LDR) | `photoresistor` | Yes | Yes |
| 10 kΩ Resistor (voltage divider) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Photoresistor | Pin 1 / Pin 2 | GPIO34 / 3V3 | Yellow / Red | Light sensor analog input |
| 10 kΩ Resistor | Leg 1 / Leg 2 | GPIO34 / GND | Black | Divider pull-down resistor |
| Shared power | VCC / GND | 3V3 / GND | Red / Black | Sensor power rails |

> **Wiring tip:** The photoresistor requires a 10 kΩ pull-down resistor to ground to form a voltage divider. This maps dark to low voltage (low raw ADC values) and light to high voltage (high raw ADC values).

## Code
```cpp
// Web Page LDR light sensor bar display (Asynchronous progress bar dashboard)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int LDR_PIN = 34;

WebServer server(80);

// 1. Root URL Handler ("/")
// Serves the HTML dashboard containing CSS, the bar, and the AJAX polling loop
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Ambient Light Monitor</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 400px; width: 100%; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; text-align: center; }\n";
  
  html += "  .metric { display: flex; justify-content: space-between; margin: 25px 0 10px 0; }\n";
  html += "  .label { font-weight: bold; color: #94a3b8; }\n";
  html += "  .value { color: #f1f5f9; font-family: monospace; font-size: 24px; font-weight: bold; }\n";
  
  // Custom Progress Bar Design
  html += "  .bar-container { background-color: #334155; height: 24px; border-radius: 12px; overflow: hidden; border: 1px solid #475569; position: relative; }\n";
  html += "  .bar-fill { height: 100%; width: 0%; background: linear-gradient(90deg, #38bdf8, #10b981); transition: width 0.3s ease; }\n";
  html += "  .bar-text { position: absolute; width: 100%; text-align: center; line-height: 24px; font-size: 12px; font-weight: bold; color: white; text-shadow: 1px 1px 2px black; }\n";
  
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Ambient Light Monitor</h1>\n";
  
  html += "  <div class=\"metric\">\n";
  html += "    <span class=\"label\">LDR Intensity:</span>\n";
  html += "    <span class=\"value\" id=\"lightPercent\">0%</span>\n";
  html += "  </div>\n";
  
  // Progress Bar
  html += "  <div class=\"bar-container\">\n";
  html += "    <div id=\"barFill\" class=\"bar-fill\"></div>\n";
  html += "    <div id=\"barText\" class=\"bar-text\">0%</div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 1 second | Port 80</p>\n";
  html += "</div>\n";
  
  // 2. Client-side AJAX polling script using Fetch API
  html += "<script>\n";
  html += "  function updateLightLevel() {\n";
  html += "    fetch('/api/light')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('lightPercent').innerText = data.percent + '%';\n";
  html += "        document.getElementById('barFill').style.width = data.percent + '%';\n";
  html += "        document.getElementById('barText').innerText = data.percent + '% (' + data.raw + ' ADC)';\n";
  html += "      })\n";
  html += "      .catch(err => console.error('Error fetching light data:', err));\n";
  html += "  }\n";
  
  // Start polling loop once page finishes loading
  html += "  window.onload = function() {\n";
  html += "    updateLightLevel();\n";
  html += "    setInterval(updateLightLevel, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// 3. API Telemetry URL Handler ("/api/light")
void handleLightAPI() {
  int ldrRaw = analogRead(LDR_PIN);
  
  // Map raw ADC to percentage (0% to 100%)
  int percent = map(ldrRaw, 0, 4095, 0, 100);
  
  // Compile JSON telemetry string
  String json = "{\"raw\":" + String(ldrRaw) + 
                 ",\"percent\":" + String(percent) + "}";
  
  server.send(200, "application/json", json);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LDR_PIN, INPUT);
  
  Serial.println("\nESP32 Light Monitor Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Register routes
  server.on("/", handleRoot);
  server.on("/api/light", handleLightAPI);
  
  server.begin();
  Serial.print("Web Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LDR** onto the canvas.
2. Wire LDR to **GPIO34**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Adjust the LDR light level slider. Watch the progress bar value and width update on the webpage.

## Expected Output
Serial Monitor:
```
ESP32 Light Monitor Server Starting...
....
WiFi Connected.
Web Server active. Connect at: http://10.10.0.3
```

Browser Console (Inspect Network tab):
* Continuous lightweight GET requests to `/api/light` returning JSON strings (`{"raw":3200,"percent":78}`).

## Expected Canvas Behavior
* Adjusting the LDR slider updates the progress bar fill and text on the webpage in real time.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `analogRead(LDR_PIN)` | Samples the analog voltage on the photoresistor pin. |
| `map(...)` | Maps the raw ADC reading to a percentage (0% to 100%). |
| `style.width = data.percent + '%'` | Modifies the CSS width property of the progress bar to show the value. |
| `server.send(200, "application/json", ...)` | Serves the light payload with the correct JSON MIME type. |

## Hardware & Safety Concept: Analog Input Filters and Divider Networks
Photoresistors change their resistance based on ambient light levels. To read this change with a microcontroller, wire the LDR with a fixed resistor to form a **voltage divider**. If the analog signal experiences noise (e.g. flickering fluorescent lights), implement a software moving-average filter to smooth the readings.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the light level.
2. **Night Light Controller**: Add a relay (GPIO 13) and turn it ON automatically if the light level drops below 20%.
3. **SPIFFS integration**: Store the webpage assets in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |
| Values display as 0.0 or nan | Divider resistor missing | Confirm that a 10 kΩ pull-down resistor is installed on the LDR pin |
| Web page responds slowly | Blocking delays | Avoid using blocking `delay()` calls inside the main loop |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [48 - ESP32 LDR light meter remote report](../beginner/48-esp32-ldr-light-meter-remote-report.md)
- [61 - ESP32 Web Server AJAX Telemetry Auto-refresh](61-esp32-web-server-ajax-telemetry-auto-refresh.md)
- [70 - ESP32 Web Page HC-SR04 distance readout](70-esp32-web-page-hc-sr04-distance-readout.md) (Next project)
