# 126 - Line Following Robot with Junction Web status

Build a junction-detecting Automated Guided Vehicle (AGV) simulation on the ESP32 that reads three photo-reflective infrared sensors (Left on GPIO 12, Middle on GPIO 13, Right on GPIO 14), drives two DC motors via an L298N driver, detects crossroad and T-junctions, and hosts a web dashboard displaying real-time junction counts and logs.

## Goal
Learn how to implement multi-sensor logic arrays, build junction-classification state machines, maintain event log queues, serve web diagnostics, and compile JSON API endpoints.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Three IR reflective line sensors are on GPIOs 12, 13, and 14, and an L298N driver on GPIOs 27, 26, 25, and 33 controls the motors. The robot follows a black line. When it encounters a junction (e.g. all three sensors on black), it identifies the junction type (Crossroad, Left T, Right T), increments a counter, and logs the event. A web page displays sensor states, total junctions passed, and recent logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| IR Reflective Line Sensors (TCRT5000) | `button` | Yes (3 pieces) | Yes (3 pieces) |
| L298N H-Bridge Motor Driver | `l298n` | Yes | Yes |
| DC Gear Motors | `motor_dc` | Yes (2 pieces) | Yes (2 pieces) |
| Shared power / Battery Pack | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use three pushbuttons to simulate the Left, Middle, and Right TCRT5000 line sensors (pressed = on line/black, unpressed = clear/white).

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| TCRT5000 Sensor (L) | OUT (Digital) | GPIO12 | Yellow | Left line sensor input |
| TCRT5000 Sensor (M) | OUT (Digital) | GPIO13 | Orange | Middle line sensor input |
| TCRT5000 Sensor (R) | OUT (Digital) | GPIO14 | Green | Right line sensor input |
| L298N Driver | IN1 / IN2 | GPIO27 / GPIO26 | Blue / White | Left motor direction |
| L298N Driver | IN3 / IN4 | GPIO25 / GPIO33 | Purple / Grey | Right motor direction |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the three TCRT5000 sensors and L298N driver from the 5V Vin rail.

## Code
```cpp
// Line Following Robot with Junction Web status (Junction Classifier + AGV Logging Node)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// TCRT5000 IR reflective line sensor pins
const int L_SENSOR_PIN = 12;
const int M_SENSOR_PIN = 13;
const int R_SENSOR_PIN = 14;

// L298N motor driver direction pins
const int IN1 = 27;
const int IN2 = 26;
const int IN3 = 25;
const int IN4 = 33;

WebServer server(80);

// Sensor state cache
bool leftOnLine = false;
bool middleOnLine = false;
bool rightOnLine = false;

int junctionCount = 0;
String currentAction = "STOP";
String lastJunctionType = "None";

// Rolling Junction Logs (Stores last 5 entries)
String junctionLogs[5] = {"", "", "", "", ""};
int logCount = 0;

void addJunctionLog(String message) {
  for (int i = 4; i > 0; i--) {
    junctionLogs[i] = junctionLogs[i - 1];
  }
  unsigned long timestampS = millis() / 1000;
  junctionLogs[0] = "[" + String(timestampS) + "s] " + message;
  
  if (logCount < 5) logCount++;
  Serial.println("[AGV] " + junctionLogs[0]);
}

// Drive control routines
void moveForward() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  currentAction = "FORWARD";
}

void turnLeft() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  currentAction = "STEER LEFT";
}

void turnRight() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  currentAction = "STEER RIGHT";
}

void stopRobot() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
  currentAction = "STOPPED";
}

// HTTP API endpoint returning JSON data
void handleGetJunctionData() {
  String json = "{\"count\":" + String(junctionCount) + 
                 ",\"action\":\"" + currentAction + "\"" +
                 ",\"lastJunction\":\"" + lastJunctionType + "\"" +
                 ",\"left\":" + String(leftOnLine ? 1 : 0) +
                 ",\"middle\":" + String(middleOnLine ? 1 : 0) +
                 ",\"right\":" + String(rightOnLine ? 1 : 0) +
                 ",\"logs\":[";
  for (int i = 0; i < 5; i++) {
    json += "\"" + junctionLogs[i] + "\"";
    if (i < 4) json += ",";
  }
  json += "]}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>AGV Telemetry HUD</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .count-val { font-size: 56px; font-weight: 800; font-family: monospace; color: #10b981; margin: 15px 0; }\n";
  html += "  .status-badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 14px; text-transform: uppercase; margin-bottom: 20px; background-color: #334155; color: #94a3b8; }\n";
  html += "  .sensor-row { display: flex; gap: 15px; justify-content: center; margin: 20px 0; }\n";
  html += "  .sensor-widget { display: flex; flex-direction: column; align-items: center; font-size: 11px; color: #64748b; }\n";
  html += "  .led-bulb { width: 20px; height: 20px; border-radius: 50%; background-color: #334155; border: 2px solid #475569; margin-top: 5px; transition: background-color 0.1s; }\n";
  html += "  .led-bulb.active { background-color: #38bdf8; box-shadow: 0 0 10px #38bdf8; }\n";
  html += "  table { width: 100%; border-collapse: collapse; margin-top: 25px; text-align: left; font-size: 13px; }\n";
  html += "  th { color: #64748b; border-bottom: 2px solid #334155; padding-bottom: 8px; }\n";
  html += "  td { padding: 8px 0; border-bottom: 1px solid #1e293b; color: #94a3b8; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>AGV Telemetry HUD</h1>\n";
  html += "  <div id=\"actionDisplay\" class=\"status-badge\">STOPPED</div>\n";
  html += "  <div id=\"countDisplay\" class=\"count-val\">0</div>\n";
  html += "  <div style=\"font-size:12px; color:#64748b; margin-top:-10px;\">Junctions Cleared</div>\n";
  
  // IR Sensors
  html += "  <div class=\"sensor-row\">\n";
  html += "    <div class=\"sensor-widget\"><span>Left</span><div id=\"leftInd\" class=\"led-bulb\"></div></div>\n";
  html += "    <div class=\"sensor-widget\"><span>Middle</span><div id=\"midInd\" class=\"led-bulb\"></div></div>\n";
  html += "    <div class=\"sensor-widget\"><span>Right</span><div id=\"rightInd\" class=\"led-bulb\"></div></div>\n";
  html += "  </div>\n";
  
  html += "  <h3>Junction History Logs</h3>\n";
  html += "  <table>\n";
  html += "    <thead><tr><th>Event History</th></tr></thead>\n";
  html += "    <tbody id=\"logTable\"></tbody>\n";
  html += "  </table>\n";
  
  html += "  <p class=\"footer\">AGV node active | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateHUD() {\n";
  html += "    fetch('/api/junction')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('countDisplay').innerText = data.count;\n";
  html += "        document.getElementById('actionDisplay').innerText = data.action;\n";
  
  html += "        document.getElementById('leftInd').className = 'led-bulb' + (data.left === 1 ? ' active' : '');\n";
  html += "        document.getElementById('midInd').className = 'led-bulb' + (data.middle === 1 ? ' active' : '');\n";
  html += "        document.getElementById('rightInd').className = 'led-bulb' + (data.right === 1 ? ' active' : '');\n";
  
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
  
  pinMode(L_SENSOR_PIN, INPUT);
  pinMode(M_SENSOR_PIN, INPUT);
  pinMode(R_SENSOR_PIN, INPUT);
  
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  
  stopRobot();
  
  Serial.println("\nESP32 AGV System starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/junction", handleGetJunctionData);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  addJunctionLog("Junction Monitor Armed");
}

void loop() {
  server.handleClient();
  
  // Read sensor inputs (HIGH when on black line, LOW when on white)
  leftOnLine = (digitalRead(L_SENSOR_PIN) == HIGH);
  middleOnLine = (digitalRead(M_SENSOR_PIN) == HIGH);
  rightOnLine = (digitalRead(R_SENSOR_PIN) == HIGH);
  
  // 1. Junction Detection State Machine
  static bool onJunctionLatch = false;
  
  bool isJunction = (leftOnLine && middleOnLine && rightOnLine) || 
                    (leftOnLine && middleOnLine && !rightOnLine) || 
                    (!leftOnLine && middleOnLine && rightOnLine);
                    
  if (isJunction) {
    if (!onJunctionLatch) {
      onJunctionLatch = true; // Lock latch to prevent multiple triggers for the same junction
      junctionCount++;
      
      // Classify junction type
      if (leftOnLine && middleOnLine && rightOnLine) {
        lastJunctionType = "CROSSROAD";
      } else if (leftOnLine && middleOnLine && !rightOnLine) {
        lastJunctionType = "LEFT T-JUNCTION";
      } else {
        lastJunctionType = "RIGHT T-JUNCTION";
      }
      
      addJunctionLog("Detected " + lastJunctionType + " (#" + String(junctionCount) + ")");
    }
  } else {
    // Release latch once the robot leaves the junction
    if (!leftOnLine && !middleOnLine && !rightOnLine) {
      onJunctionLatch = false;
    }
  }
  
  // 2. Normal Line Following Control Loop
  if (leftOnLine && !rightOnLine) {
    turnLeft();
  } 
  else if (!leftOnLine && rightOnLine) {
    turnRight();
  } 
  else if (middleOnLine) {
    moveForward();
  } 
  else {
    // If track is completely lost, stop the robot
    stopRobot();
    currentAction = "TRACK LOST - STOPPED";
  }
  
  delay(5);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **3 Pushbuttons** (to simulate Left, Middle, and Right sensors), **L298N Driver**, and **2 DC Motors** onto the canvas.
2. Wire Left/Middle/Right Buttons to **GPIO12/GPIO13/GPIO14**, and IN1–IN4 to **GPIO27, 26, 25, and 33**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. With the Middle button pressed, verify that both simulated motors spin forward (green clockwise/counterclockwise).
6. Press the Left and Middle buttons together (simulating a Left T-Junction). Verify that the Junction counter increments to 1, and the web logs record `Detected LEFT T-JUNCTION`.
7. Release all buttons. Verify that the motors stop.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[AGV] [0s] Junction Monitor Armed
[AGV] [15s] Detected LEFT T-JUNCTION (#1)
[AGV] [24s] Detected CROSSROAD (#2)
```

## Expected Canvas Behavior
* Pressing combinations of the three sensor buttons triggers junction classification logic.
* The web page updates the junction count and sensor bulbs in real time.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `isJunction` | Check condition: returns true if multiple sensors hit the line. |
| `onJunctionLatch = true` | Latches the junction state to prevent double counts. |
| `lastJunctionType = "CROSSROAD"` | Identifies the junction layout based on sensor states. |
| `turnLeft()` / `turnRight()` | Standard steering commands to keep the robot centered on the line. |

## Hardware & Safety Concept: Junction Latching and Track Lost recovery
* **Junction Latching**: Because robot wheels turn relatively slowly, the sensors will stay over a junction for several execution loops (each loop takes 5 ms). Without a **latching flag** (`onJunctionLatch`), the code would count a single junction dozens of times. Setting a latch until all sensors leave the black area prevents this.
* **Track Lost Recovery**: If a robot misses a turn and drives off the line entirely (all sensors on white), it must shut down its motors immediately (`stopRobot()`) to prevent it from driving off the table.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a symbol showing the junction type.
2. **Alert chime buzzer**: Sound a brief beep on a buzzer (GPIO 15) when a junction is passed.
3. **SPIFFS integration**: Save the total junction count to SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Junction counter jumps by 2 or more | Latch reset too early | Adjust the latch reset condition to ensure all sensors have cleared the line |
| Robot spins instead of turning | Motor direction wrong | Swap the output wires on the motor driver for that motor |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [49 - Potentiometer LED Bar Graph level controller](../beginner/49-potentiometer-led-bar-graph-level-controller.md)
- [124 - ESP32 Obstacle Avoidance Robot with Web HUD](124-esp32-obstacle-avoidance-robot-with-web-hud-telemetry.md)
- [127 - ESP32 MQTT Remote Control Robot](127-esp32-mqtt-remote-control-robot-publish-navigation-keys.md) (Next project)
