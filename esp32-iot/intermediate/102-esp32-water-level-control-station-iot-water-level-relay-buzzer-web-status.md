# 102 - Water Level Control Station IoT (Water level + Relay + Buzzer + Web status)

Build an automated water level controller on the ESP32 that samples an analog water sensor on GPIO 34, triggers a pump relay on GPIO 13 and an alarm buzzer on GPIO 15 when levels drop below 20%, shuts down the pump when levels reach 80%, and hosts a web dashboard with real-time telemetry.

## Goal
Learn how to sample analog water levels, implement hysteresis logic control loops, integrate multiple warnings, serve interactive web dashboards, and package JSON APIs.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A water level sensor is on GPIO 34, a buzzer on GPIO 15, and a relay on GPIO 13. The ESP32 hosts a webpage displaying water level percentages, status flags, and pump activity. If the level drops below 20%, the buzzer sounds an alarm, the relay closes (to turn on a water pump), and the web dashboard updates. The pump stays ON until water levels exceed 80%.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Water Level Sensor | `potentiometer` | Yes | Yes |
| Passive Buzzer Module | `buzzer` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |

> **MbedO Note:** In simulation, use a potentiometer to simulate the analog output of the water level sensor.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Water Sensor | OUT (Analog) | GPIO34 | Yellow | Water depth signal |
| Passive Buzzer | VCC (+) | GPIO15 | Blue | Warning buzzer |
| Relay Module | IN (Signal) | GPIO13 | Orange | Water pump relay |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the sensor, buzzer, and relay from the 5V Vin rail.

## Code
```cpp
// Water Level Control Station IoT (Level depth telemetry + Hysteresis Pump Control)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int WATER_PIN = 34;
const int BUZZER_PIN = 15;
const int RELAY_PIN = 13;

WebServer server(80);

int waterPercent = 0;
bool pumpActive = false;

// Return status string based on level and pump state
String getStatusString() {
  if (waterPercent < 20) return "WARNING: CRITICAL LOW";
  if (waterPercent >= 80) return "TANK FULL";
  return "NORMAL";
}

// HTTP API endpoint returning JSON data
void handleGetWaterData() {
  String json = "{\"water\":" + String(waterPercent) + 
                 ",\"status\":\"" + getStatusString() + "\"" +
                 ",\"pump\":" + String(pumpActive ? 1 : 0) + "}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Water Tank Dashboard</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 400px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  
  // Dynamic Tank Fill Graphic Design
  html += "  .tank-container { width: 120px; height: 180px; border: 4px solid #334155; border-radius: 8px 8px 12px 12px; margin: 25px auto; position: relative; overflow: hidden; background-color: #0f172a; }\n";
  html += "  .water-fill { position: absolute; bottom: 0; left: 0; width: 100%; background: linear-gradient(180deg, #38bdf8, #0284c7); transition: height 0.5s ease-in-out; }\n";
  html += "  .tank-text { position: absolute; width: 100%; text-align: center; top: 45%; font-size: 24px; font-weight: bold; font-family: monospace; color: white; text-shadow: 1px 1px 3px black; z-index: 10; }\n";
  
  html += "  .badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 14px; text-transform: uppercase; margin-bottom: 10px; }\n";
  html += "  .badge.normal { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .badge.warning { background-color: #991b1b; color: #fee2e2; animation: pulse 1s infinite; }\n";
  html += "  .badge.full { background-color: #0369a1; color: #e0f2fe; }\n";
  
  html += "  .pump-status { font-size: 16px; color: #94a3b8; margin-top: 15px; }\n";
  html += "  .pump-status.active { color: #10b981; font-weight: bold; }\n";
  html += "  @keyframes pulse { 0%, 100% { opacity: 1; } 50% { opacity: 0.5; } }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Water Level Console</h1>\n";
  html += "  <div id=\"waterBadge\" class=\"badge normal\">NORMAL</div>\n";
  
  // Visual Tank
  html += "  <div class=\"tank-container\">\n";
  html += "    <div id=\"tankVal\" class=\"tank-text\">0%</div>\n";
  html += "    <div id=\"waterFill\" class=\"water-fill\" style=\"height: 0%;\"></div>\n";
  html += "  </div>\n";
  
  html += "  <div id=\"pumpDisplay\" class=\"pump-status\">Water Pump: OFF</div>\n";
  html += "  <p class=\"footer\">Asynchronous updates polling every 1 second | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateTankLevel() {\n";
  html += "    fetch('/api/water')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('tankVal').innerText = data.water + '%';\n";
  html += "        document.getElementById('waterFill').style.height = data.water + '%';\n";
  
  html += "        const bdgEl = document.getElementById('waterBadge');\n";
  html += "        const pmpEl = document.getElementById('pumpDisplay');\n";
  
  html += "        bdgEl.innerText = data.status;\n";
  
  html += "        if (data.water < 20) {\n";
  html += "          bdgEl.className = 'badge warning';\n";
  html += "        } else if (data.water >= 80) {\n";
  html += "          bdgEl.className = 'badge full';\n";
  html += "        } else {\n";
  html += "          bdgEl.className = 'badge normal';\n";
  html += "        }\n";
  
  html += "        if (data.pump === 1) {\n";
  html += "          pmpEl.innerText = 'Water Pump: ACTIVE (FILLING)';\n";
  html += "          pmpEl.className = 'pump-status active';\n";
  html += "        } else {\n";
  html += "          pmpEl.innerText = 'Water Pump: OFF';\n";
  html += "          pmpEl.className = 'pump-status';\n";
  html += "        }\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateTankLevel();\n";
  html += "    setInterval(updateTankLevel, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(WATER_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(RELAY_PIN, OUTPUT);
  
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(RELAY_PIN, LOW); // Start with pump off
  
  Serial.println("\nESP32 Water Station Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/water", handleGetWaterData);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // Read analog water depth sensor
  int rawVal = analogRead(WATER_PIN);
  waterPercent = map(rawVal, 0, 4095, 0, 100);
  
  // 1. Enforce Hysteresis Control Logic
  if (waterPercent < 20) {
    // Critical Low -> Activate Pump Relay and sound Buzzer
    pumpActive = true;
    digitalWrite(RELAY_PIN, HIGH);
    
    // Pulse alarm tone on buzzer
    static unsigned long lastTone = 0;
    if (millis() - lastTone >= 300) {
      static bool soundOn = false;
      soundOn = !soundOn;
      if (soundOn) tone(BUZZER_PIN, 1000); else noTone(BUZZER_PIN);
      lastTone = millis();
    }
  } 
  else if (waterPercent >= 80) {
    // Tank Full -> Deactivate Pump Relay and silence Buzzer
    pumpActive = false;
    digitalWrite(RELAY_PIN, LOW);
    noTone(BUZZER_PIN);
  } 
  else {
    // In-between states: keep pump running if it was already active
    noTone(BUZZER_PIN); // Silence the buzzer once it's above 20%
    if (pumpActive) {
      digitalWrite(RELAY_PIN, HIGH);
    } else {
      digitalWrite(RELAY_PIN, LOW);
    }
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer** (to simulate the water sensor), **Buzzer**, and **Relay** onto the canvas.
2. Wire Potentiometer to **GPIO34**, Buzzer to **GPIO15**, and Relay to **GPIO13**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Adjust the potentiometer slider.
5. Move the potentiometer slider below 20%. Verify that the Buzzer sounds, the Relay turns ON (pump active), and the web dashboard displays a flashing warning.
6. Slowly slide the potentiometer upwards. Note that the pump stays ON (hysteresis range) until the slider exceeds 80%, at which point the pump shuts OFF automatically.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
```

Browser JSON Console (`/api/water`):
```json
{"water":15,"status":"WARNING: CRITICAL LOW","pump":1}
```

## Expected Canvas Behavior
* Adjusting the potentiometer simulates water level changes.
* The relay and buzzer widgets actuate based on the hysteresis logic threshold checks (turning ON below 20%, turning OFF above 80%).

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `analogRead(WATER_PIN)` | Samples the analog voltage on the depth sensor pin. |
| `waterPercent < 20` | Critical low boundary checking trigger point. |
| `waterPercent >= 80` | High level shut-off boundary checking trigger point. |
| `digitalWrite(RELAY_PIN, ...)` | Closes/opens the relay contacts to turn the pump ON/OFF. |

## Hardware & Safety Concept: Hysteresis Loops and Pump Protection
* **Hysteresis Control**: If a single threshold (e.g. 50%) is used to turn the pump ON and OFF, the pump will cycle rapidly ON and OFF whenever the water level ripples around 50%. This can burn out the pump's motor coils. Enforcing a **hysteresis loop** (ON below 20%, OFF above 80%) prevents this rapid cycling.
* **Dry-Run Protection**: Water pumps must not run when dry as this can destroy the pump impeller. Install a physical float switch in series with the relay coil to cut off power to the pump if the intake well goes dry.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a fill bar graph.
2. **Dynamic fill alarm**: Sound a chime when the tank reaches 100% full to alert the operator.
3. **SPIFFS integration**: Log daily pump run times to a file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer does not sound | Buzzer pin misconfigured | Confirm that the passive buzzer is connected to GPIO 15 |
| Pump does not switch | Relay signal wired wrong | Confirm that the relay is wired to GPIO 13 |
| Water values display as 0% | Potentiometer wired wrong | Verify that the wiper pin is connected to GPIO 34 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [46 - ESP32 Web-based Relay switch control](../beginner/46-esp32-web-based-relay-switch-control.md)
- [98 - ESP32 Gas Leakage Alarm IoT Node](98-esp32-gas-leakage-alarm-iot-node-mq-2-buzzer-relay-web-alert.md)
- [103 - ESP32 Smart Doorbell Alert IoT](103-esp32-smart-doorbell-alert-iot-pir-chime-lcd-web-notification.md) (Next project)
