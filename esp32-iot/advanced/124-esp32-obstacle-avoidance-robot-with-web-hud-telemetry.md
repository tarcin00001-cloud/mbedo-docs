# 124 - Obstacle Avoidance Robot with Web HUD telemetry

Build an autonomous obstacle avoidance vehicle on the ESP32 that measures distances using an HC-SR04 ultrasonic range-finder (Trig on GPIO 12, Echo on GPIO 14), drives two DC motors via an L298N driver, and hosts a web dashboard displaying real-time distance telemetry and evasion logs.

## Goal
Learn how to capture pulse timing from ultrasonic sensors, implement non-blocking autonomous obstacle avoidance state machines, serve web dashboards, and compile JSON API endpoints.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An HC-SR04 ultrasonic sensor is on GPIO 12/14, and an L298N driver on GPIOs 27, 26, 25, and 33 controls two DC motors. The robot drives forward autonomously. If it detects an obstacle closer than 20 cm, it automatically stops, reverses, turns right to find a clear path, and logs the event. A web page displays live distance readings and a history table of evasion events.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| L298N H-Bridge Motor Driver | `l298n` | Yes | Yes |
| DC Gear Motors | `motor_dc` | Yes (2 pieces) | Yes (2 pieces) |
| Shared power / Battery Pack | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 Sensor | TRIG | GPIO12 | Yellow | Trigger pulse output |
| HC-SR04 Sensor | ECHO | GPIO14 | Green | Echo pulse input |
| L298N Driver | IN1 / IN2 | GPIO27 / GPIO26 | Blue / White | Left motor control |
| L298N Driver | IN3 / IN4 | GPIO25 / GPIO33 | Purple / Grey | Right motor control |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the HC-SR04 sensor and L298N driver from the 5V Vin rail.

## Code
```cpp
// Obstacle Avoidance Robot with Web HUD telemetry (Autonomous Evasion + Web telemetry)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// HC-SR04 Pins
const int TRIG_PIN = 12;
const int ECHO_PIN = 14;

// L298N motor driver pins
const int IN1 = 27;
const int IN2 = 26;
const int IN3 = 25;
const int IN4 = 33;

WebServer server(80);

float currentDistanceCm = 100.0;
String robotState = "FORWARD";

// Evasion state variables
enum DriveState { STATE_FORWARD, STATE_BACKING_UP, STATE_TURNING };
DriveState currentState = STATE_FORWARD;
unsigned long stateTimer = 0;

// Rolling Evasion Logs (Stores last 5 entries)
String evasionLogs[5] = {"", "", "", "", ""};
int logCount = 0;

void addEvasionLog(String message) {
  for (int i = 4; i > 0; i--) {
    evasionLogs[i] = evasionLogs[i - 1];
  }
  unsigned long timestampS = millis() / 1000;
  evasionLogs[0] = "[" + String(timestampS) + "s] " + message;
  
  if (logCount < 5) logCount++;
  Serial.println("[ROBOT] " + evasionLogs[0]);
}

// Read HC-SR04 sensor distance in cm
float readDistanceCm() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  long duration = pulseIn(ECHO_PIN, HIGH, 30000); // 30ms timeout (max 5 meters)
  if (duration == 0) return 400.0; // Return max distance if out of range
  
  return (float)duration * 0.0343 / 2.0;
}

// Drive control routines
void moveForward() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  robotState = "FORWARD";
}

void moveBackward() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  robotState = "REVERSING";
}

void turnRight() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  robotState = "EVADING RIGHT";
}

void stopRobot() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
  robotState = "STOPPED";
}

// HTTP API endpoint returning JSON data
void handleGetTelemetry() {
  String json = "{\"distance\":" + String(currentDistanceCm, 1) + 
                 ",\"state\":\"" + robotState + "\"" +
                 ",\"logs\":[";
  for (int i = 0; i < 5; i++) {
    json += "\"" + evasionLogs[i] + "\"";
    if (i < 4) json += ",";
  }
  json += "]}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Robot Telemetry HUD</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .distance-val { font-size: 56px; font-weight: 800; font-family: monospace; color: #10b981; margin: 20px 0; }\n";
  html += "  .distance-val.warning { color: #f59e0b; }\n";
  html += "  .distance-val.critical { color: #ef4444; animation: pulse 0.5s infinite; }\n";
  html += "  .status-badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 14px; text-transform: uppercase; margin-bottom: 20px; background-color: #334155; color: #94a3b8; }\n";
  html += "  .status-badge.evading { background-color: #991b1b; color: #fee2e2; }\n";
  html += "  table { width: 100%; border-collapse: collapse; margin-top: 25px; text-align: left; font-size: 13px; }\n";
  html += "  th { color: #64748b; border-bottom: 2px solid #334155; padding-bottom: 8px; }\n";
  html += "  td { padding: 8px 0; border-bottom: 1px solid #1e293b; color: #94a3b8; }\n";
  html += "  @keyframes pulse { 0%, 100% { opacity: 1; } 50% { opacity: 0.5; } }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Obstacle Avoidance HUD</h1>\n";
  html += "  <div id=\"stateBadge\" class=\"status-badge\">FORWARD</div>\n";
  html += "  <div id=\"distDisplay\" class=\"distance-val\">100.0 cm</div>\n";
  
  html += "  <h3>Evasion Logs</h3>\n";
  html += "  <table>\n";
  html += "    <thead><tr><th>Event History</th></tr></thead>\n";
  html += "    <tbody id=\"logTable\"></tbody>\n";
  html += "  </table>\n";
  
  html += "  <p class=\"footer\">Telemetry console active | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateHUD() {\n";
  html += "    fetch('/api/telemetry')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('distDisplay').innerText = data.distance.toFixed(1) + ' cm';\n";
  
  html += "        const valEl = document.getElementById('distDisplay');\n";
  html += "        const bdgEl = document.getElementById('stateBadge');\n";
  html += "        bdgEl.innerText = data.state;\n";
  
  html += "        if (data.distance < 20) {\n";
  html += "          valEl.className = 'distance-val critical';\n";
  html += "          bdgEl.className = 'status-badge evading';\n";
  html += "        } else if (data.distance < 50) {\n";
  html += "          valEl.className = 'distance-val warning';\n";
  html += "          bdgEl.className = 'status-badge';\n";
  html += "        } else {\n";
  html += "          valEl.className = 'distance-val';\n";
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
  
  html += "  window.onload = function() {\n";
  html += "    updateHUD();\n";
  html += "    setInterval(updateHUD, 500); // Poll twice a second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  
  stopRobot();
  
  Serial.println("\nESP32 Autonomous Robot starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/telemetry", handleGetTelemetry);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  addEvasionLog("Autonomous System Initialized");
}

void loop() {
  server.handleClient();
  
  // 1. Measure distance in cm
  currentDistanceCm = readDistanceCm();
  
  // 2. Non-blocking state machine navigation control
  unsigned long now = millis();
  
  switch (currentState) {
    case STATE_FORWARD:
      if (currentDistanceCm < 20.0) {
        // Obstacle detected! Reverse immediately
        stopRobot();
        addEvasionLog("Obstacle at " + String(currentDistanceCm, 1) + "cm! Reversing...");
        moveBackward();
        currentState = STATE_BACKING_UP;
        stateTimer = now;
      } else {
        moveForward();
      }
      break;
      
    case STATE_BACKING_UP:
      // Reverse for 800 ms to clear path
      if (now - stateTimer >= 800) {
        stopRobot();
        addEvasionLog("Clearing... Turning Right...");
        turnRight();
        currentState = STATE_TURNING;
        stateTimer = now;
      }
      break;
      
    case STATE_TURNING:
      // Turn right for 600 ms to rotate away
      if (now - stateTimer >= 600) {
        stopRobot();
        addEvasionLog("Path cleared. Proceeding forward.");
        currentState = STATE_FORWARD;
      }
      break;
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **HC-SR04 Sensor**, **L298N Driver**, and **2 DC Motors** onto the canvas.
2. Wire TRIG/ECHO to **GPIO12/GPIO14**, and IN1–IN4 to **GPIO27, 26, 25, and 33**. Wire the motor outputs of the L298N to the motors.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. In normal state, the ultrasonic distance is set high (e.g. 100 cm). Verify that the motor widgets spin forward (green clockwise/counterclockwise).
6. Decrease the simulated ultrasonic distance below 20 cm. Verify that the motor widgets stop, reverse, turn right, and return to forward navigation, and the event displays in the web logs.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[ROBOT] [0s] Autonomous System Initialized
[ROBOT] [15s] Obstacle at 15.4cm! Reversing...
[ROBOT] [16s] Clearing... Turning Right...
[ROBOT] [17s] Path cleared. Proceeding forward.
```

## Expected Canvas Behavior
* Adjusting the simulated distance slider below 20 cm stops, reverses, and spins the motor widgets.
* The web telemetry page reflects the distance values and evasion logs.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `readDistanceCm()` | Triggers the HC-SR04 and calculates the distance in cm based on echo pulse length. |
| `currentState = STATE_BACKING_UP` | Moves the non-blocking state machine into the backup state. |
| `now - stateTimer >= 800` | Timer condition to transition states without blocking the web server. |
| `turnRight()` | Drives the left motor forward and right motor backward to rotate the robot. |

## Hardware & Safety Concept: Sensor Read Failures and Non-blocking Execution
* **Sensor Read Failures**: If an ultrasonic sensor wire disconnects, `pulseIn()` will time out and return 0. If the code does not handle this, the robot might interpret 0 cm as a close obstacle and get stuck spinning in circles forever. Treat 0 cm as an out-of-range value (e.g. mapping to 400 cm) to prevent this lockup.
* **Non-blocking execution**: Avoid using `delay()` inside autonomous control loops. Doing so freezes the web server, causing the web dashboard to lose connection. Always use state machine variables and `millis()` timers.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a radar graphic showing distance.
2. **Reverse buzzer alarm**: Add a buzzer (GPIO 15) to sound a warning beep when the robot reverses.
3. **Double Sensor Radar**: Add a second ultrasonic sensor to scan left and right paths before turning.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Distance always reads 0 cm | Echo pin misconfigured | Confirm that the Echo pin is connected to an input pin (GPIO 14) and Trig to an output pin (GPIO 12) |
| Robot gets stuck reversing | Timer logic issue | Check if `stateTimer` is updated when transitioning states in the state machine |
| Webpage console lags | Poll rate too fast | Increase the polling interval on the webpage script (e.g. to 500 ms or 1000 ms) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [50 - Potentiometer servo angle controller](../beginner/50-potentiometer-servo-angle-controller-bar-display.md)
- [112 - ESP32 DS3231 Real Time Clock Web Sync Display](112-esp32-ds3231-real-time-clock-web-sync-display.md)
- [125 - ESP32 Line Following Robot with Web Configuration](125-esp32-line-following-robot-with-web-configuration-dashboard.md) (Next project)
