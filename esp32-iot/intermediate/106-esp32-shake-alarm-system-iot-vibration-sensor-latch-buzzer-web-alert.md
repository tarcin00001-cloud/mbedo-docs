# 106 - Shake Alarm System IoT (Vibration sensor + Latch + Buzzer + Web alert)

Build a latched anti-tamper alarm system on the ESP32 that detects shakes or vibrations using a sensor on GPIO 14, locks into an active alarm state, sounds a pulsing alarm on a passive buzzer (GPIO 15), and hosts a web dashboard with manual reset override options and event logs.

## Goal
Learn how to capture rapid digital signals, implement latching alarm state loops, write debounce routines for reset inputs, design web consoles, and process HTTP POST override requests.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A vibration sensor is on GPIO 14, a buzzer on GPIO 15, and a reset button on GPIO 12. If a shake is detected, the ESP32 locks into an alarm state, pulsing the buzzer continuously. The buzzer stays active until the reset button is pressed or a reset command is sent from the web dashboard.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Vibration Sensor (SW-420) | `button` | Yes | Yes |
| Pushbutton (Reset) | `button` | Yes | Yes |
| Passive Buzzer Module | `buzzer` | Yes | Yes |

> **MbedO Note:** In simulation, use a pushbutton configured as active-HIGH (external pull-down or code mapping) to simulate the vibration sensor output. Use a second button as active-LOW (internal pull-up) for the Reset button.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Vibration Sensor | OUT (Signal) | GPIO14 | Yellow | Tamper trigger input |
| Reset Button | Pin 1 / Pin 2 | GPIO12 / GND | Blue / Black | Manual reset button |
| Passive Buzzer | VCC (+) | GPIO15 | Green | Siren audio output |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the vibration sensor and buzzer from the 5V Vin rail.

## Code
```cpp
// Shake Alarm System IoT (Vibration trigger latch + Buzzer siren + Web Reset)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int VIBRATION_PIN = 14;
const int RESET_PIN = 12;
const int BUZZER_PIN = 15;

WebServer server(80);

// Alarm states
bool alarmLatched = false;
unsigned long lastSirenToggle = 0;
bool sirenSoundOn = false;

// Rolling logs buffer (Stores last 5 entries)
String alarmLogs[5] = {"", "", "", "", ""};
int logCount = 0;

void addAlarmLog(String message) {
  for (int i = 4; i > 0; i--) {
    alarmLogs[i] = alarmLogs[i - 1];
  }
  unsigned long timestampS = millis() / 1000;
  alarmLogs[0] = "[" + String(timestampS) + "s] " + message;
  
  if (logCount < 5) logCount++;
  Serial.println("[SECURITY] " + alarmLogs[0]);
}

// Reset alarm state
void resetAlarm() {
  if (alarmLatched) {
    alarmLatched = false;
    noTone(BUZZER_PIN);
    addAlarmLog("Alarm System Reset");
  }
}

// HTTP API endpoint returning JSON data
void handleGetAlarmData() {
  String json = "{\"alarm\":" + String(alarmLatched ? 1 : 0) + ",\"logs\":[";
  for (int i = 0; i < 5; i++) {
    json += "\"" + alarmLogs[i] + "\"";
    if (i < 4) json += ",";
  }
  json += "]}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to reset alarm
void handlePostReset() {
  resetAlarm();
  server.send(200, "text/plain", "OK");
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Security Alarm Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .status-badge { display: inline-block; padding: 10px 20px; border-radius: 20px; font-weight: bold; font-size: 16px; text-transform: uppercase; margin: 20px 0; }\n";
  html += "  .secured { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .tampered { background-color: #991b1b; color: #fee2e2; animation: flash 1s infinite; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 14px; font-size: 16px; font-weight: bold; color: white; background-color: #ef4444; border: none; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; margin-top: 15px; }\n";
  html += "  .btn:hover { background-color: #dc2626; }\n";
  html += "  table { width: 100%; border-collapse: collapse; margin-top: 25px; text-align: left; font-size: 13px; }\n";
  html += "  th { color: #64748b; border-bottom: 2px solid #334155; padding-bottom: 8px; }\n";
  html += "  td { padding: 8px 0; border-bottom: 1px solid #1e293b; color: #94a3b8; }\n";
  html += "  @keyframes flash { 0%, 100% { opacity: 1; } 50% { opacity: 0.5; } }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Security Alarm Console</h1>\n";
  html += "  <div id=\"alarmBadge\" class=\"status-badge secured\">SECURED</div>\n";
  
  html += "  <button class=\"btn\" onclick=\"resetAlarm()\">Reset Alarm Latch</button>\n";
  
  html += "  <h3>Security Activity Logs</h3>\n";
  html += "  <table>\n";
  html += "    <thead><tr><th>Event History</th></tr></thead>\n";
  html += "    <tbody id=\"logTable\"></tbody>\n";
  html += "  </table>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 500 ms | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateAlarmStatus() {\n";
  html += "    fetch('/api/alarm')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        const bdgEl = document.getElementById('alarmBadge');\n";
  
  html += "        if (data.alarm === 1) {\n";
  html += "          bdgEl.innerText = 'WARNING: TAMPER DETECTED';\n";
  html += "          bdgEl.className = 'status-badge tampered';\n";
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
  html += "      .then(() => updateAlarmStatus());\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateAlarmStatus();\n";
  html += "    setInterval(updateAlarmStatus, 500); // Poll twice a second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(VIBRATION_PIN, INPUT);
  pinMode(RESET_PIN, INPUT_PULLUP);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("\nESP32 Security Alarm Station Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/alarm", handleGetAlarmData);
  server.on("/api/reset", HTTP_POST, handlePostReset);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  addAlarmLog("Alarm System Armed");
}

void loop() {
  server.handleClient();
  
  // Read vibration sensor (HIGH when vibration is active)
  bool vibrationDetected = (digitalRead(VIBRATION_PIN) == HIGH);
  
  // 1. Alarm Latching Logic
  // Lock the system into the active state on the first detection
  if (vibrationDetected && !alarmLatched) {
    alarmLatched = true;
    addAlarmLog("Tamper Alert: Vibration Detected!");
  }
  
  // 2. Local physical reset button handler
  if (digitalRead(RESET_PIN) == LOW) {
    delay(50); // Simple debounce delay
    if (digitalRead(RESET_PIN) == LOW) {
      resetAlarm();
    }
  }
  
  // 3. Siren tone scheduler (Runs only when alarm is latched)
  if (alarmLatched) {
    unsigned long now = millis();
    if (now - lastSirenToggle >= 200) { // Alternating tones every 200 ms
      sirenSoundOn = !sirenSoundOn;
      if (sirenSoundOn) {
        tone(BUZZER_PIN, 1200); // High-pitched siren
      } else {
        tone(BUZZER_PIN, 800);  // Low-pitched siren
      }
      lastSirenToggle = now;
    }
  } else {
    noTone(BUZZER_PIN);
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Pushbutton** (configured as active-HIGH to simulate the vibration sensor), **Pushbutton** (configured as active-LOW for Reset), and **Buzzer** onto the canvas.
2. Wire Vibration Button to **GPIO14**, Reset Button to **GPIO12**, and Buzzer to **GPIO15**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Click the vibration button widget once. Verify that the buzzer starts sounding its siren continuously, and the web dashboard logs the tamper warning.
6. Click the reset button widget or click the "Reset Alarm Latch" button on the webpage. Verify that the buzzer stops.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[SECURITY] [0s] Alarm System Armed
[SECURITY] [6s] Tamper Alert: Vibration Detected!
[SECURITY] [12s] Alarm System Reset
```

## Expected Canvas Behavior
* Triggering the simulated vibration button locks the buzzer widget into a pulsing state immediately.
* Pressing the physical reset button or web override button silences the buzzer.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `vibrationDetected && !alarmLatched` | Latches the alarm state on the first detected shake. |
| `digitalRead(RESET_PIN) == LOW` | Checks the active-LOW reset button. |
| `tone(BUZZER_PIN, ...)` | Generates alternating frequencies on the buzzer pin to sound the siren. |
| `server.on("/api/reset", ...)` | Registers the API endpoint to reset the alarm latch from the web dashboard. |

## Hardware & Safety Concept: Latched Alarms and Tamper Detection
In security systems, alarm triggers must be **latched** (locked in). If an intruder shakes a security box and breaks the sensor, a non-latched alarm would silence immediately, allowing the intruder to escape unnoticed. Locking the alarm state ensures the siren sounds continuously until an authorized operator resets the system.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a warning graphic when the alarm is latched.
2. **Warning LED Strobe**: Add a red LED on GPIO 13 to flash in sync with the siren.
3. **SPIFFS integration**: Log alarm trigger statistics to a file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers continuously | Floating input pin | Confirm that the vibration sensor has a pull-down resistor or uses correct digital configuration |
| Reset button does not work | Debounce error | Check the pull-up wiring on the reset button pin (GPIO 12) |
| Web page responds slowly | Blocking loops | Ensure that there are no blocking delay functions inside the main loop |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [28 - ESP32 Wireless limit switch reporter](../beginner/28-esp32-wireless-limit-switch-reporter.md)
- [95 - ESP-NOW Broadcast warning siren](95-esp-now-broadcast-warning-siren.md)
- [107 - ESP32 Fire Warning Station IoT](107-esp32-fire-warning-station-iot-flame-buzzer-oled-web-telemetry.md) (Next project)
