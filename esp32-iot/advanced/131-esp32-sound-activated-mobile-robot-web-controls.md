# 131 - Sound Activated Mobile Robot Web controls

Build a sound-triggered mobile robot on the ESP32 that toggles its drive state (Forward/Stop) when it detects loud sounds (like claps) using an analog sound sensor on GPIO 34, drives two DC motors via an L298N driver, and hosts a web dashboard with threshold sliders and event logs.

## Goal
Learn how to capture acoustic signal pulses, implement software debounce routines for sound triggers, write toggle state machines, serve web dashboards, and process HTTP POST configuration updates.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An analog sound sensor is on GPIO 34, and an L298N driver on GPIOs 27, 26, 25, and 33 controls the motors. A clap toggles the robot between driving forward and stopping. Navigating to the ESP32's IP address displays a dashboard showing the live sound level, the robot's state, and a slider to configure the clap detection threshold.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Analog Sound Sensor | `potentiometer` | Yes | Yes |
| L298N H-Bridge Motor Driver | `l298n` | Yes | Yes |
| DC Gear Motors | `motor_dc` | Yes (2 pieces) | Yes (2 pieces) |
| Shared power / Battery Pack | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use a potentiometer to simulate the analog output of the sound sensor. Turning it up quickly simulates a loud clap sound.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Sound Sensor | OUT (Analog) | GPIO34 | Yellow | Sound amplitude signal |
| L298N Driver | IN1 / IN2 | GPIO27 / GPIO26 | Blue / White | Left motor control |
| L298N Driver | IN3 / IN4 | GPIO25 / GPIO33 | Purple / Grey | Right motor control |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the sound sensor and L298N driver from the 5V Vin rail.

## Code
```cpp
// Sound Activated Mobile Robot Web controls (Clap Toggle Control + Web Threshold Configuration)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int SOUND_PIN = 34;

// L298N motor driver direction pins
const int IN1 = 27;
const int IN2 = 26;
const int IN3 = 25;
const int IN4 = 33;

WebServer server(80);

int soundLevel = 0;
int triggerThreshold = 1800; // Default analog threshold (0-4095)
bool isMoving = false;

unsigned long lastTriggerTime = 0;
const unsigned long DEBOUNCE_TIME_MS = 500; // Prevent double trigger within 500 ms

// Rolling Event Logs (Stores last 5 entries)
String acousticLogs[5] = {"", "", "", "", ""};
int logCount = 0;

void addAcousticLog(String message) {
  for (int i = 4; i > 0; i--) {
    acousticLogs[i] = acousticLogs[i - 1];
  }
  unsigned long timestampS = millis() / 1000;
  acousticLogs[0] = "[" + String(timestampS) + "s] " + message;
  
  if (logCount < 5) logCount++;
  Serial.println("[SOUND] " + acousticLogs[0]);
}

// Drive control routines
void moveForward() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  isMoving = true;
}

void stopRobot() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
  isMoving = false;
}

// HTTP API endpoint returning JSON data
void handleGetStatus() {
  String json = "{\"sound\":" + String(soundLevel) + 
                 ",\"threshold\":" + String(triggerThreshold) +
                 ",\"moving\":" + String(isMoving ? 1 : 0) +
                 ",\"logs\":[";
  for (int i = 0; i < 5; i++) {
    json += "\"" + acousticLogs[i] + "\"";
    if (i < 4) json += ",";
  }
  json += "]}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to update threshold
void handlePostThreshold() {
  if (server.hasArg("threshold")) {
    triggerThreshold = server.arg("threshold").toInt();
    Serial.printf("[Scheduler] Threshold updated to %d\n", triggerThreshold);
    server.send(200, "text/plain", "OK");
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Sound Control Dashboard</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .status-badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 14px; text-transform: uppercase; margin-bottom: 20px; background-color: #334155; color: #94a3b8; }\n";
  html += "  .status-badge.active { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .sound-val { font-size: 56px; font-weight: 800; font-family: monospace; color: #38bdf8; margin: 15px 0; }\n";
  html += "  .slider-container { margin: 25px 0; text-align: left; }\n";
  html += "  label { font-weight: bold; color: #94a3b8; display: block; margin-bottom: 12px; }\n";
  html += "  input[type=range] { width: 100%; height: 8px; border-radius: 4px; background: #334155; outline: none; -webkit-appearance: none; }\n";
  html += "  input[type=range]::-webkit-slider-thumb { -webkit-appearance: none; width: 22px; height: 22px; border-radius: 50%; background: #38bdf8; cursor: pointer; }\n";
  html += "  table { width: 100%; border-collapse: collapse; margin-top: 25px; text-align: left; font-size: 13px; }\n";
  html += "  th { color: #64748b; border-bottom: 2px solid #334155; padding-bottom: 8px; }\n";
  html += "  td { padding: 8px 0; border-bottom: 1px solid #1e293b; color: #94a3b8; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Sound Control Console</h1>\n";
  html += "  <div id=\"stateBadge\" class=\"status-badge\">STOPPED</div>\n";
  html += "  <div id=\"soundDisplay\" class=\"sound-val\">0</div>\n";
  
  html += "  <div class=\"slider-container\">\n";
  html += "    <label for=\"threshRange\">Acoustic Threshold: <span id=\"threshVal\">1800</span></label>\n";
  html += "    <input type=\"range\" id=\"threshRange\" min=\"500\" max=\"3500\" value=\"" + String(triggerThreshold) + "\" oninput=\"updateThreshold(this.value)\">\n";
  html += "  </div>\n";
  
  html += "  <h3>Acoustic Activity Logs</h3>\n";
  html += "  <table>\n";
  html += "    <thead><tr><th>Event History</th></tr></thead>\n";
  html += "    <tbody id=\"logTable\"></tbody>\n";
  html += "  </table>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 200 ms | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateHUD() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('soundDisplay').innerText = data.sound;\n";
  
  html += "        const bdgEl = document.getElementById('stateBadge');\n";
  html += "        if (data.moving === 1) {\n";
  html += "          bdgEl.innerText = 'DRIVING FORWARD';\n";
  html += "          bdgEl.className = 'status-badge active';\n";
  html += "        } else {\n";
  html += "          bdgEl.innerText = 'STOPPED';\n";
  html += "          bdgEl.className = 'status-badge';\n";
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
  
  html += "  function updateThreshold(val) {\n";
  html += "    document.getElementById('threshVal').innerText = val;\n";
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('threshold', val);\n";
  html += "    fetch('/api/threshold', { method: 'POST', body: body });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateHUD();\n";
  html += "    setInterval(updateHUD, 200); // Fast poll for live sound monitoring\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(SOUND_PIN, INPUT);
  
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  
  stopRobot();
  
  Serial.println("\nESP32 Sound Control Robot starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/status", handleGetStatus);
  server.on("/api/threshold", HTTP_POST, handlePostThreshold);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  addAcousticLog("Acoustic Sensor Armed");
}

void loop() {
  server.handleClient();
  
  // 1. Read sound sensor amplitude value
  soundLevel = analogRead(SOUND_PIN);
  unsigned long now = millis();
  
  // 2. Sound Trigger Toggle Logic
  if (soundLevel > triggerThreshold) {
    // Debounce check
    if (now - lastTriggerTime >= DEBOUNCE_TIME_MS) {
      lastTriggerTime = now;
      
      // Toggle drive state
      if (isMoving) {
        stopRobot();
        addAcousticLog("Loud Sound Detected (" + String(soundLevel) + "). Toggled state to STOPPED.");
      } else {
        moveForward();
        addAcousticLog("Loud Sound Detected (" + String(soundLevel) + "). Toggled state to FORWARD.");
      }
    }
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer** (to simulate the sound sensor), **L298N Driver**, and **2 DC Motors** onto the canvas.
2. Wire Potentiometer to **GPIO34**, and IN1–IN4 to **GPIO27, 26, 25, and 33**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. In normal state, keep the potentiometer slider low. Verify that the robot stays stopped.
6. Turn the potentiometer slider quickly above 2000 (simulating a clap) and then slide it back down. Verify that the simulated motors turn ON, the state changes to `DRIVING FORWARD`, and the web history logs the toggle event.
7. Repeat the action to stop the motors.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[SOUND] [0s] Acoustic Sensor Armed
[SOUND] [12s] Loud Sound Detected (2150). Toggled state to FORWARD.
[SOUND] [18s] Loud Sound Detected (2410). Toggled state to STOPPED.
```

## Expected Canvas Behavior
* Sliding the potentiometer slider quickly above the threshold toggles motor widget rotation ON and OFF.
* The web page logs the transition state.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `analogRead(SOUND_PIN)` | Samples the sound sensor output voltage. |
| `soundLevel > triggerThreshold` | Trigger comparison checking if the sound level exceeds the threshold. |
| `now - lastTriggerTime >= DEBOUNCE_TIME_MS` | Prevents double triggers from the same sound. |
| `isMoving = !isMoving` | Toggles the active drive state. |

## Hardware & Safety Concept: Sound Debouncing and Ambient Noise Filtering
* **Debouncing Sound**: A single clap creates sound waves that ripple for 200–300 ms, causing the analog reading to spike multiple times. Without a **debounce timer** (`DEBOUNCE_TIME_MS`), the code would register these ripples as multiple claps, toggling the state rapidly.
* **Ambient Noise Filtering**: In noisy environments, normal conversation can trigger the sensor. Use the webpage slider to adjust the threshold above the ambient noise level, ensuring the robot only triggers on distinct, loud claps.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a microphone icon.
2. **Double Clap Turn**: Implement double-clap detection: a single clap toggles start/stop, while two quick claps within 1 second execute a right turn.
3. **SPIFFS integration**: Log sound trigger history logs to a file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot triggers on normal sounds | Threshold too low | Increase the threshold slider value on the webpage |
| Robot does not respond to claps | Threshold too high | Decrease the threshold slider value on the webpage |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [46 - ESP32 Web-based Relay switch control](../beginner/46-esp32-web-based-relay-switch-control.md)
- [96 - ESP32 Web-controlled Sound Level OLED DB meter](96-esp32-web-controlled-sound-level-oled-db-meter.md)
- [132 - ESP32 Servo Robotic Arm Web Slider controls](132-esp32-servo-robotic-arm-web-slider-controls.md) (Next project)
