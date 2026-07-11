# 68 - Web Page MQ-2 Gas sensor gauge display

Build a web-connected safety monitor on the ESP32 that hosts a control page, reads gas concentration values from an MQ-2 sensor on GPIO 34, serves a JSON API endpoint, and uses client-side JavaScript (AJAX polling) to refresh a gas leakage severity index gauge every second.

## Goal
Learn how to sample gas sensors, map raw ADC inputs to percentages, categorize security states, implement client-side AJAX polling loops, and design visual warning gauges.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a web server on port 80. An MQ-2 gas sensor is on GPIO 34. Navigating to the ESP32's IP address displays a webpage with a styled gauge indicating gas levels (0% to 100%). JavaScript on the page polls `/api/gas` every second, updating the gauge and a safety status label (Safe, Warning, or Danger) immediately.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MQ-2 Gas Sensor Module | `potentiometer` | Yes | Yes |

> **MbedO Note:** In simulation, use a potentiometer wired to GPIO 34 to simulate the analog output of the MQ-2 gas sensor.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MQ-2 Gas Sensor | AO (Analog Out) | GPIO34 | Yellow | Gas density signal |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Sensor power rails |

> **Wiring tip:** Connect the analog output of the MQ-2 sensor directly to GPIO 34. Power the sensor from the 5V Vin rail.

## Code
```cpp
// Web Page MQ-2 Gas sensor gauge display (Asynchronous safety gauge dashboard)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int GAS_PIN = 34;

WebServer server(80);

// 1. Root URL Handler ("/")
// Serves the HTML dashboard containing CSS, the gauge, and the AJAX polling loop
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Gas Leakage Monitor</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 350px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; margin-bottom: 25px; }\n";
  
  // Custom Gauge Design
  html += "  .gauge-container { position: relative; width: 160px; height: 160px; margin: 0 auto 25px auto; }\n";
  html += "  .gauge-bg { width: 100%; height: 100%; border-radius: 50%; background: conic-gradient(#475569 0deg, #475569 360deg); display: flex; justify-content: center; align-items: center; transition: background 0.3s; }\n";
  html += "  .gauge-inner { width: 120px; height: 120px; border-radius: 50%; background-color: #1e293b; display: flex; flex-direction: column; justify-content: center; align-items: center; }\n";
  html += "  .gauge-val { font-size: 32px; font-weight: 800; font-family: monospace; }\n";
  
  html += "  .status-badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 14px; text-transform: uppercase; margin-top: 10px; }\n";
  html += "  .safe { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .warning { background-color: #92400e; color: #fef3c7; }\n";
  html += "  .danger { background-color: #991b1b; color: #fee2e2; animation: blink 1s infinite; }\n";
  
  html += "  @keyframes blink { 0%, 100% { opacity: 1; } 50% { opacity: 0.5; } }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Gas Leakage Monitor</h1>\n";
  
  // Visual Gauge
  html += "  <div class=\"gauge-container\">\n";
  html += "    <div id=\"gaugeBg\" class=\"gauge-bg\">\n";
  html += "      <div class=\"gauge-inner\">\n";
  html += "        <div id=\"gasPercent\" class=\"gauge-val\">0%</div>\n";
  html += "        <div id=\"gasRaw\" style=\"font-size: 12px; color: #64748b; margin-top: 5px;\">ADC: 0</div>\n";
  html += "      </div>\n";
  html += "    </div>\n";
  html += "  </div>\n";
  
  // Status Badge
  html += "  <div id=\"statusLabel\" class=\"status-badge safe\">SAFE</div>\n";
  html += "  <p class=\"footer\">Asynchronous updates polling every 1 second | Port 80</p>\n";
  html += "</div>\n";
  
  // 2. Client-side AJAX polling script using Fetch API
  html += "<script>\n";
  html += "  function updateGasLevel() {\n";
  html += "    fetch('/api/gas')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('gasPercent').innerText = data.percent + '%';\n";
  html += "        document.getElementById('gasRaw').innerText = 'ADC: ' + data.raw;\n";
  
  // Update gauge gradient color and angle
  // Map percentage (0-100) to degrees (0-360)
  html += "        const angle = (data.percent / 100) * 360;\n";
  html += "        const bg = document.getElementById('gaugeBg');\n";
  html += "        const lbl = document.getElementById('statusLabel');\n";
  
  html += "        if (data.percent < 30) {\n";
  html += "          bg.style.background = 'conic-gradient(#10b981 ' + angle + 'deg, #475569 ' + angle + 'deg)';\n";
  html += "          lbl.className = 'status-badge safe';\n";
  html += "          lbl.innerText = 'SAFE';\n";
  html += "        } else if (data.percent >= 30 && data.percent < 60) {\n";
  html += "          bg.style.background = 'conic-gradient(#f59e0b ' + angle + 'deg, #475569 ' + angle + 'deg)';\n";
  html += "          lbl.className = 'status-badge warning';\n";
  html += "          lbl.innerText = 'WARNING';\n";
  html += "        } else {\n";
  html += "          bg.style.background = 'conic-gradient(#ef4444 ' + angle + 'deg, #475569 ' + angle + 'deg)';\n";
  html += "          lbl.className = 'status-badge danger';\n";
  html += "          lbl.innerText = 'DANGER: LEAK DETECTED';\n";
  html += "        }\n";
  html += "      })\n";
  html += "      .catch(err => console.error('Error fetching gas data:', err));\n";
  html += "  }\n";
  
  // Start polling loop once page finishes loading
  html += "  window.onload = function() {\n";
  html += "    updateGasLevel();\n";
  html += "    setInterval(updateGasLevel, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// 3. API Telemetry URL Handler ("/api/gas")
void handleGasAPI() {
  int gasRaw = analogRead(GAS_PIN);
  
  // Map raw ADC to percentage (0% to 100%)
  int percent = map(gasRaw, 0, 4095, 0, 100);
  
  // Compile JSON telemetry string
  String json = "{\"raw\":" + String(gasRaw) + 
                 ",\"percent\":" + String(percent) + "}";
  
  server.send(200, "application/json", json);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(GAS_PIN, INPUT);
  
  Serial.println("\nESP32 Gas Monitor Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Register routes
  server.on("/", handleRoot);
  server.on("/api/gas", handleGasAPI);
  
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
1. Drag **ESP32 DevKitC** and **Potentiometer** onto the canvas.
2. Wire Potentiometer to **GPIO34**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Slide the potentiometer widget. Watch the gauge value, color, and warning badge update on the webpage.

## Expected Output
Serial Monitor:
```
ESP32 Gas Monitor Server Starting...
....
WiFi Connected.
Web Server active. Connect at: http://10.10.0.3
```

Browser Console (Inspect Network tab):
* Continuous lightweight GET requests to `/api/gas` returning JSON strings (`{"raw":1200,"percent":29}`).

## Expected Canvas Behavior
* Adjusting the potentiometer updates the gauge percentage and color (Green -> Yellow -> Red) on the webpage in real time.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `analogRead(GAS_PIN)` | Samples the analog voltage on the gas sensor pin. |
| `conic-gradient(...)` | CSS style to draw a circular progress ring based on the active angle. |
| `setInterval(updateGasLevel, 1000)` | Registers a browser timer to call the update function every 1000 ms. |
| `server.send(200, "application/json", ...)` | Serves the gas payload with the correct JSON MIME type. |

## Hardware & Safety Concept: Gas Sensor Calibration and Warnings
MQ-2 gas sensors require a warm-up time (often up to 24 hours in hardware) before readings stabilize. Additionally, raw ADC readings depend on ambient humidity and air quality. Map raw readings to percentages and define clear warning boundaries (e.g. Danger > 60%) to prevent false alarms.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the gas level.
2. **Gas leakage buzzer alarm**: Add a buzzer (GPIO 15) and sound an alarm if the level exceeds 60%.
3. **SPIFFS integration**: Store the webpage assets in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |
| Values display as 0.0 or nan | Sensor connection error | Confirm that the sensor data pin is connected to GPIO 34 |
| Web page responds slowly | Blocking delays | Avoid using blocking `delay()` calls inside the main loop |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [61 - ESP32 Web Server AJAX Telemetry Auto-refresh](61-esp32-web-server-ajax-telemetry-auto-refresh.md)
- [67 - ESP32 Web Page DHT22 climate display](67-esp32-web-page-dht22-climate-display.md)
- [69 - ESP32 Web Page LDR light sensor bar display](69-esp32-web-page-ldr-light-sensor-bar-display.md) (Next project)
