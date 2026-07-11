# 110 - Soil Moisture Automated Irrigation IoT (Soil sensor + Relay + Web logger)

Build an automated smart irrigation control node on the ESP32 that samples an analog resistive soil moisture sensor on GPIO 34, triggers a water pump relay on GPIO 13 when soil moisture drops below 35%, shuts down the pump once moisture exceeds 70%, and hosts a web dashboard with real-time logging.

## Goal
Learn how to sample analog soil moisture levels, implement hysteresis logic control loops, serve interactive web dashboards, and compile JSON API endpoints with rolling logs.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A soil moisture sensor is on GPIO 34, and a water pump relay on GPIO 13. The ESP32 hosts a webpage displaying moisture levels, system states, and pump activity. If the moisture level drops below 35%, the relay closes (to turn on the pump) and logs the event. The pump stays ON until the soil moisture exceeds 70%.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Soil Moisture Sensor | `potentiometer` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use a potentiometer to simulate the analog output of the soil moisture sensor.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Soil Sensor | AO (Analog Out) | GPIO34 | Yellow | Moisture index signal |
| Relay Module | IN (Signal) | GPIO13 | Orange | Water pump relay |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the soil moisture sensor and relay module from the 5V Vin rail.

## Code
```cpp
// Soil Moisture Automated Irrigation IoT (Moisture telemetry + Hysteresis Pump Control + Web logs)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int SOIL_PIN = 34;
const int RELAY_PIN = 13;

WebServer server(80);

int soilMoisture = 0;
bool pumpActive = false;

// Rolling logs database (Stores last 5 measurements)
String irrigationLogs[5] = {"", "", "", "", ""};
int logCount = 0;

void addIrrigationLog(String message) {
  for (int i = 4; i > 0; i--) {
    irrigationLogs[i] = irrigationLogs[i - 1];
  }
  unsigned long timestampS = millis() / 1000;
  irrigationLogs[0] = "[" + String(timestampS) + "s] " + message;
  
  if (logCount < 5) logCount++;
  Serial.println("[IRRIGATION] " + irrigationLogs[0]);
}

// Get active operating status
String getStatusString() {
  if (soilMoisture < 35) return "DRY - IRRIGATION REQUIRED";
  if (soilMoisture >= 70) return "WET - TANK SATURATED";
  return "NORMAL";
}

// HTTP API endpoint returning JSON data
void handleGetIrrigationData() {
  String json = "{\"moisture\":" + String(soilMoisture) + 
                 ",\"status\":\"" + getStatusString() + "\"" +
                 ",\"pump\":" + String(pumpActive ? 1 : 0) + "\"" +
                 ",\"logs\":[";
  for (int i = 0; i < 5; i++) {
    json += "\"" + irrigationLogs[i] + "\"";
    if (i < 4) json += ",";
  }
  json += "]}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Smart Irrigation Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .moisture-val { font-size: 64px; font-weight: 800; font-family: monospace; color: #38bdf8; margin: 20px 0; }\n";
  html += "  .moisture-val.dry { color: #f59e0b; }\n";
  html += "  .badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 14px; text-transform: uppercase; margin-bottom: 20px; }\n";
  html += "  .badge.normal { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .badge.dry { background-color: #92400e; color: #fef3c7; }\n";
  html += "  .badge.wet { background-color: #0369a1; color: #e0f2fe; }\n";
  html += "  .pump-status { font-size: 16px; color: #94a3b8; margin-top: 15px; }\n";
  html += "  .pump-status.active { color: #10b981; font-weight: bold; }\n";
  html += "  table { width: 100%; border-collapse: collapse; margin-top: 25px; text-align: left; font-size: 13px; }\n";
  html += "  th { color: #64748b; border-bottom: 2px solid #334155; padding-bottom: 8px; }\n";
  html += "  td { padding: 8px 0; border-bottom: 1px solid #1e293b; color: #94a3b8; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Smart Irrigation Console</h1>\n";
  html += "  <div id=\"statusBadge\" class=\"badge normal\">NORMAL</div>\n";
  html += "  <div id=\"moistureDisplay\" class=\"moisture-val\">0%</div>\n";
  html += "  <div id=\"pumpDisplay\" class=\"pump-status\">Water Pump: OFF</div>\n";
  
  html += "  <h3>Irrigation Activity Logs</h3>\n";
  html += "  <table>\n";
  html += "    <thead><tr><th>Logs Database</th></tr></thead>\n";
  html += "    <tbody id=\"logTable\"></tbody>\n";
  html += "  </table>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 1 second | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateIrrigation() {\n";
  html += "    fetch('/api/irrigation')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('moistureDisplay').innerText = data.moisture + '%';\n";
  
  html += "        const valEl = document.getElementById('moistureDisplay');\n";
  html += "        const bdgEl = document.getElementById('statusBadge');\n";
  html += "        const pmpEl = document.getElementById('pumpDisplay');\n";
  
  html += "        bdgEl.innerText = data.status;\n";
  
  html += "        if (data.moisture < 35) {\n";
  html += "          bdgEl.className = 'badge dry';\n";
  html += "          valEl.className = 'moisture-val dry';\n";
  html += "        } else if (data.moisture >= 70) {\n";
  html += "          bdgEl.className = 'badge wet';\n";
  html += "          valEl.className = 'moisture-val';\n";
  html += "        } else {\n";
  html += "          bdgEl.className = 'badge normal';\n";
  html += "          valEl.className = 'moisture-val';\n";
  html += "        }\n";
  
  html += "        if (data.pump === 1) {\n";
  html += "          pmpEl.innerText = 'Water Pump: ACTIVE';\n";
  html += "          pmpEl.className = 'pump-status active';\n";
  html += "        } else {\n";
  html += "          pmpEl.innerText = 'Water Pump: OFF';\n";
  html += "          pmpEl.className = 'pump-status';\n";
  html += "        }\n";
  
  html += "        const tableBody = document.getElementById('logTable');\n";
  html += "        tableBody.innerHTML = '';\n";
  html += "        data.logs.forEach(log => {\n";
  html += "          if (log.length > 0) {\n";
  html += "            tableBody.innerHTML += '<tr><td>' + log + '</td></tr>';\n";
  html += "          }\n";
  html += "        });\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateIrrigation();\n";
  html += "    setInterval(updateIrrigation, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(SOIL_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start with pump off
  
  Serial.println("\nESP32 Irrigation Node Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/irrigation", handleGetIrrigationData);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  addIrrigationLog("Irrigation Controller Armed");
}

void loop() {
  server.handleClient();
  
  // Read analog soil moisture sensor
  // Soil moisture sensors output lower voltages when wet (0V at maximum moisture, 3.3V when dry)
  int rawVal = analogRead(SOIL_PIN);
  soilMoisture = map(rawVal, 4095, 0, 0, 100);
  if (soilMoisture < 0) soilMoisture = 0;
  if (soilMoisture > 100) soilMoisture = 100;
  
  // Enforce Hysteresis Control Logic
  if (soilMoisture < 35) {
    if (!pumpActive) {
      pumpActive = true;
      digitalWrite(RELAY_PIN, HIGH); // Turn pump ON
      addIrrigationLog("Soil Dry (" + String(soilMoisture) + "%). Pump turned ON.");
    }
  } 
  else if (soilMoisture >= 70) {
    if (pumpActive) {
      pumpActive = false;
      digitalWrite(RELAY_PIN, LOW);  // Turn pump OFF
      addIrrigationLog("Soil Hydrated (" + String(soilMoisture) + "%). Pump turned OFF.");
    }
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer** (to simulate the soil moisture sensor), and **Relay** onto the canvas.
2. Wire Potentiometer to **GPIO34** and Relay to **GPIO13**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Adjust the potentiometer slider.
5. Move the potentiometer slider to simulate dry soil (lower the value to simulate water density). Verify that the Relay turns ON (pump active) and the web dashboard logs the event.
6. Slowly slide the potentiometer upwards. Note that the pump stays ON (hysteresis range) until the slider exceeds 70%, at which point the pump shuts OFF automatically.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[IRRIGATION] [0s] Irrigation Controller Armed
[IRRIGATION] [10s] Soil Dry (28%). Pump turned ON.
[IRRIGATION] [18s] Soil Hydrated (72%). Pump turned OFF.
```

Browser JSON Console (`/api/irrigation`):
```json
{"moisture":25,"status":"DRY - IRRIGATION REQUIRED","pump":1,"logs":["...","..."]}
```

## Expected Canvas Behavior
* Adjusting the potentiometer simulates soil moisture changes.
* The relay widget turns ON when the simulated moisture drops below 35%, and turns OFF when it exceeds 70%.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `map(rawVal, 4095, 0, 0, 100)` | Inverts and maps the soil moisture sensor output (0V is wet, 3.3V is dry). |
| `soilMoisture < 35` | Critical dry threshold trigger point. |
| `soilMoisture >= 70` | Soil hydration limit trigger point. |
| `digitalWrite(RELAY_PIN, ...)` | Closes/opens the relay contacts to turn the pump ON/OFF. |

## Hardware & Safety Concept: Hysteresis loops and Corrosion Prevention
* **Hysteresis loops**: If the pump turned ON at 40% and OFF at 41%, the water level would trigger rapid pump cycling, which would quickly destroy the pump. Set a wide hysteresis window (e.g. 35% to 70%) to protect the pump.
* **Corrosion Prevention**: Resistive soil sensors pass current through the soil to measure conductivity. Keeping them powered continuously causes rapid electrolytic corrosion, destroying the probes within weeks. To prevent this, power the sensor from a GPIO pin and turn it ON only during sampling, keeping it OFF the rest of the time.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a watering icon.
2. **Rain Sensor Override**: Add a rain sensor (Project 99) and disable the pump if it is raining.
3. **SPIFFS integration**: Log daily watering volume to a file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Pump does not switch | Relay signal pin miswired | Verify that the relay signal is connected to GPIO 13 |
| Moisture readings are always 0% | Potentiometer wired wrong | Verify that the wiper pin is connected to GPIO 34 |
| Webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [46 - ESP32 Web-based Relay switch control](../beginner/46-esp32-web-based-relay-switch-control.md)
- [102 - ESP32 Water Level Control Station IoT](102-esp32-water-level-control-station-iot-water-level-relay-buzzer-web-status.md)
- [109 - ESP32 Analog Scale Reader IoT](109-esp32-analog-scale-reader-iot-load-cell-hx711-lcd-web-database.md)
