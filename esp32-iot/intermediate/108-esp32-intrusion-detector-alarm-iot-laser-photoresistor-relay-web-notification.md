# 108 - Intrusion Detector Alarm IoT (Laser + Photoresistor + Relay + Web notification)

Build an automated laser break-beam security barrier on the ESP32 that controls a laser transmitter on GPIO 12, samples an analog LDR light sensor on GPIO 34, triggers a latched siren relay on GPIO 13 when the beam is broken, and hosts a web dashboard with manual reset overrides and event logs.

## Goal
Learn how to actuate laser emitters, sample LDR photoresistors, implement break-beam detection logic, write latching alarm state loops, serve web dashboards, and process HTTP POST configurations.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A laser emitter is on GPIO 12, an LDR sensor on GPIO 34, and a siren relay on GPIO 13. On boot, the ESP32 turns the laser ON. The laser shines directly on the LDR. If someone walks through and breaks the laser beam, the LDR voltage drops, locking the system into an alarm state, closing the relay, and logging the entry. The alarm remains active until reset via the web page.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Laser Transmitter Module | `led` | Yes | Yes |
| Photoresistor (LDR) | `photoresistor` | Yes | Yes |
| 10 kΩ Resistor (voltage divider) | `resistor` | No | Yes |
| Relay Module | `relay` | Yes | Yes |

> **MbedO Note:** In simulation, use a standard Red LED to simulate the laser transmitter. Adjust the LDR light level widget to simulate aligning and breaking the beam.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Laser Module | Pin (+) | GPIO12 | Red | Laser emitter output |
| Photoresistor | Pin 1 / Pin 2 | GPIO34 / 3V3 | Yellow / Red | LDR analog input |
| 10 kΩ Resistor | Leg 1 / Leg 2 | GPIO34 / GND | Black | Divider pull-down resistor |
| Relay Module | IN (Signal) | GPIO13 | Orange | Siren control relay |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the laser, LDR divider network, and relay from the 5V Vin rail.

## Code
```cpp
// Intrusion Detector Alarm IoT (Laser beam control + LDR break beam latch + Web Reset)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int LASER_PIN = 12;
const int LDR_PIN = 34;
const int RELAY_PIN = 13;

WebServer server(80);

int ldrPercent = 0;
bool beamBroken = false;
bool alarmLatched = false;

// Rolling logs buffer (Stores last 5 entries)
String securityLogs[5] = {"", "", "", "", ""};
int logCount = 0;

void addSecurityLog(String message) {
  for (int i = 4; i > 0; i--) {
    securityLogs[i] = securityLogs[i - 1];
  }
  unsigned long timestampS = millis() / 1000;
  securityLogs[0] = "[" + String(timestampS) + "s] " + message;
  
  if (logCount < 5) logCount++;
  Serial.println("[SECURITY] " + securityLogs[0]);
}

// Reset alarm state
void resetSecurityAlarm() {
  if (alarmLatched) {
    alarmLatched = false;
    digitalWrite(RELAY_PIN, LOW); // Open relay (Siren OFF)
    addSecurityLog("Security System Reset");
  }
}

// HTTP API endpoint returning JSON data
void handleGetSecurityData() {
  String beamState = beamBroken ? "BROKEN" : "ALIGNED";
  String alarmState = alarmLatched ? "TRIPPED" : "SECURED";
  
  String json = "{\"ldr\":" + String(ldrPercent) + 
                 ",\"beam\":\"" + beamState + "\"" +
                 ",\"alarm\":\"" + alarmState + "\"" +
                 ",\"logs\":[";
  for (int i = 0; i < 5; i++) {
    json += "\"" + securityLogs[i] + "\"";
    if (i < 4) json += ",";
  }
  json += "]}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to reset alarm
void handlePostReset() {
  resetSecurityAlarm();
  server.send(200, "text/plain", "OK");
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Laser Security Dashboard</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 20px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; }\n";
  html += "  .metric-label { font-size: 12px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 24px; font-weight: bold; margin-top: 5px; font-family: monospace; }\n";
  html += "  .status-badge { display: inline-block; padding: 10px 20px; border-radius: 20px; font-weight: bold; font-size: 16px; text-transform: uppercase; margin: 15px 0; width: 100%; box-sizing: border-box; }\n";
  html += "  .secured { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .tripped { background-color: #991b1b; color: #fee2e2; animation: flash 1s infinite; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 14px; font-size: 16px; font-weight: bold; color: white; background-color: #38bdf8; border: none; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; }\n";
  html += "  .btn:hover { background-color: #0ea5e9; }\n";
  html += "  table { width: 100%; border-collapse: collapse; margin-top: 25px; text-align: left; font-size: 13px; }\n";
  html += "  th { color: #64748b; border-bottom: 2px solid #334155; padding-bottom: 8px; }\n";
  html += "  td { padding: 8px 0; border-bottom: 1px solid #1e293b; color: #94a3b8; }\n";
  html += "  @keyframes flash { 0%, 100% { opacity: 1; } 50% { opacity: 0.5; } }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Laser Security Console</h1>\n";
  html += "  <div id=\"alarmBadge\" class=\"status-badge secured\">SECURED</div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Laser Beam</div><div class=\"metric-val\" id=\"beamDisplay\">ALIGNED</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">LDR Intensity</div><div class=\"metric-val\" id=\"ldrDisplay\">0%</div></div>\n";
  html += "  </div>\n";
  
  html += "  <button class=\"btn\" onclick=\"resetAlarm()\">Reset Alarm Latch</button>\n";
  
  html += "  <h3>Intrusion Log History</h3>\n";
  html += "  <table>\n";
  html += "    <thead><tr><th>Event History</th></tr></thead>\n";
  html += "    <tbody id=\"logTable\"></tbody>\n";
  html += "  </table>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 500 ms | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateSecurityStatus() {\n";
  html += "    fetch('/api/security')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('ldrDisplay').innerText = data.ldr + '%';\n";
  html += "        document.getElementById('beamDisplay').innerText = data.beam;\n";
  html += "        document.getElementById('beamDisplay').style.color = data.beam === 'BROKEN' ? '#ef4444' : '#f1f5f9';\n";
  
  html += "        const bdgEl = document.getElementById('alarmBadge');\n";
  html += "        if (data.alarm === 'TRIPPED') {\n";
  html += "          bdgEl.innerText = 'WARNING: INTRUSION!';\n";
  html += "          bdgEl.className = 'status-badge tripped';\n";
  html += "        } else {\n";
  html += "          bdgEl.innerText = 'SYSTEM SECURED';\n";
  html += "          bdgEl.className = 'status-badge secured';\n";
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
  
  html += "  function resetAlarm() {\n";
  html += "    fetch('/api/reset', { method: 'POST' })\n";
  html += "      .then(() => updateSecurityStatus());\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateSecurityStatus();\n";
  html += "    setInterval(updateSecurityStatus, 500); // Poll twice a second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LASER_PIN, OUTPUT);
  pinMode(LDR_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  
  digitalWrite(LASER_PIN, HIGH); // Turn ON laser emitter
  digitalWrite(RELAY_PIN, LOW);   // Start with relay OPEN (off)
  
  Serial.println("\nESP32 Laser Barrier Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/security", handleGetSecurityData);
  server.on("/api/reset", HTTP_POST, handlePostReset);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  addSecurityLog("Laser Barrier Armed");
}

void loop() {
  server.handleClient();
  
  // Read LDR intensity (higher values indicate laser is aligned)
  int rawVal = analogRead(LDR_PIN);
  ldrPercent = map(rawVal, 0, 4095, 0, 100);
  
  // A threshold of 45% separates aligned vs broken states
  beamBroken = (ldrPercent < 45);
  
  // Latch alarm if the beam is broken
  if (beamBroken && !alarmLatched) {
    alarmLatched = true;
    digitalWrite(RELAY_PIN, HIGH); // Close relay (Siren ON)
    addSecurityLog("Intrusion Alert: Laser Beam Broken!");
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **LED** (to simulate the laser emitter), **Potentiometer** (to simulate the LDR), and **Relay** onto the canvas.
2. Wire LED to **GPIO12**, Potentiometer to **GPIO34**, and Relay to **GPIO13**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. In normal state, the LDR value is high (e.g. slide the potentiometer above 50%). Verify that the system stays "Secured".
6. Slide the potentiometer below 40% (simulating someone breaking the beam). Verify that the Relay turns ON (siren active) and stays ON even if you return the potentiometer to the high position.
7. Click the "Reset Alarm Latch" button on the webpage. Verify that the relay shuts OFF.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[SECURITY] [0s] Laser Barrier Armed
[SECURITY] [8s] Intrusion Alert: Laser Beam Broken!
[SECURITY] [14s] Security System Reset
```

## Expected Canvas Behavior
* Sliding the potentiometer down to simulate a broken beam turns the relay widget ON.
* The relay stays active until the reset override command is triggered from the webpage.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `digitalWrite(LASER_PIN, HIGH)` | Keeps the laser emitter powered to focus light onto the LDR. |
| `beamBroken = (ldrPercent < 45)` | Determines if the beam has been broken. |
| `alarmLatched = true` | Latches the alarm state on the first detection. |
| `digitalWrite(RELAY_PIN, HIGH)` | Closes the relay contacts to turn the siren ON. |

## Hardware & Safety Concept: Optical Alignment and Laser Eye Safety
* **Alignment**: Lasers require precise physical alignment with the LDR receiver. To simplify alignment in hardware, place the LDR inside a black cardboard tube to block ambient light and focus the laser beam directly onto the sensor.
* **Laser Safety**: Never use high-power lasers that can cause eye damage. Always use low-power Class 1 or Class 2 laser modules (under 1 mW) that are safe for short exposures.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a padlock icon showing if the alarm is armed.
2. **Warning buzzer siren**: Add a buzzer (GPIO 15) to sound an alternating siren when the alarm is tripped.
3. **SPIFFS integration**: Log alarm trigger statistics to a file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers continuously | LDR misaligned | Verify that the laser is focused directly onto the LDR and LDR reading is above 50% |
| Relay does not switch | Relay signal pin miswired | Verify that the relay signal is connected to GPIO 13 |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [30 - Laser receiver wireless gate trigger](../beginner/30-laser-receiver-wireless-gate-trigger.md)
- [106 - ESP32 Shake Alarm System IoT](106-esp32-shake-alarm-system-iot-vibration-sensor-latch-buzzer-web-alert.md)
- [109 - ESP32 Analog Scale Reader IoT](109-esp32-analog-scale-reader-iot-load-cell-hx711-lcd-web-database.md) (Next project)
