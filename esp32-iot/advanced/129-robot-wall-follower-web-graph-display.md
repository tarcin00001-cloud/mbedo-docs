# 129 - Robot Wall Follower Web Graph display

Build an autonomous wall-following robot on the ESP32 that measures wall distances using a side-mounted HC-SR04 ultrasonic range-finder (Trig on GPIO 12, Echo on GPIO 14), drives two DC motors via an L298N driver, and hosts a web dashboard with a real-time scrolling HTML5 Canvas line chart of distance history.

## Goal
Learn how to implement threshold-based feedback loops, manage circular history buffers, render dynamic line charts on HTML5 Canvas, serve web pages, and build JSON API endpoints.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An HC-SR04 ultrasonic sensor is on GPIO 12/14, and an L298N driver on GPIOs 27, 26, 25, and 33 controls the motors. The robot follows a wall on its right side at a target distance of 15 cm. If it gets too close (< 12 cm), it steers left; if it gets too far (> 18 cm), it steers right. A webpage displays a scrolling line graph of the last 20 distance readings.

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
// Robot Wall Follower Web Graph display (Autonomous Wall Tracker + HTML5 Canvas Plotter)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// HC-SR04 Pins (Right side-mounted)
const int TRIG_PIN = 12;
const int ECHO_PIN = 14;

// L298N motor driver direction pins
const int IN1 = 27;
const int IN2 = 26;
const int IN3 = 25;
const int IN4 = 33;

WebServer server(80);

float currentDistanceCm = 15.0;
String robotAction = "STOP";

// Circular buffer to store last 20 readings for graphing
const int HISTORY_SIZE = 20;
float distanceHistory[HISTORY_SIZE];
int historyIndex = 0;

void addHistoryReading(float val) {
  distanceHistory[historyIndex] = val;
  historyIndex = (historyIndex + 1) % HISTORY_SIZE;
}

// Read HC-SR04 sensor distance in cm
float readDistanceCm() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  long duration = pulseIn(ECHO_PIN, HIGH, 30000);
  if (duration == 0) return 100.0;
  
  return (float)duration * 0.0343 / 2.0;
}

// Drive control routines
void moveForward() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  robotAction = "FORWARD";
}

void steerLeft() {
  // Pivot away from the right wall
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  robotAction = "STEER LEFT";
}

void steerRight() {
  // Pivot toward the right wall
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  robotAction = "STEER RIGHT";
}

void stopRobot() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
  robotAction = "STOPPED";
}

// HTTP API endpoint returning JSON data
void handleGetTelemetry() {
  String json = "{\"distance\":" + String(currentDistanceCm, 1) + 
                 ",\"action\":\"" + robotAction + "\"" +
                 ",\"history\":[";
  for (int i = 0; i < HISTORY_SIZE; i++) {
    // Read oldest reading first (circular buffer alignment)
    int idx = (historyIndex + i) % HISTORY_SIZE;
    json += String(distanceHistory[idx], 1);
    if (i < HISTORY_SIZE - 1) json += ",";
  }
  json += "]}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Wall Follower HUD</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 480px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .status-badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 14px; text-transform: uppercase; margin-bottom: 20px; background-color: #334155; color: #94a3b8; }\n";
  html += "  .status-badge.adjusting { background-color: #0369a1; color: #e0f2fe; }\n";
  html += "  canvas { background-color: #0f172a; border-radius: 8px; border: 1px solid #334155; margin-top: 15px; width: 100%; aspect-ratio: 2/1; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Wall Follower Telemetry</h1>\n";
  html += "<div id=\"actionDisplay\" class=\"status-badge\">STOPPED</div>\n";
  
  html += "  <div style=\"font-size:24px; font-weight:bold; font-family:monospace; margin-bottom:5px;\" id=\"distVal\">--.- cm</div>\n";
  
  // Real-time canvas
  html += "  <canvas id=\"chartCanvas\" width=\"400\" height=\"200\"></canvas>\n";
  
  html += "  <p class=\"footer\">Telemetry updates polling every 200 ms | Target: 15.0 cm</p>\n";
  html += "</div>\n";
  
  // JavaScript canvas drawing script
  html += "<script>\n";
  html += "  const canvas = document.getElementById('chartCanvas');\n";
  html += "  const ctx = canvas.getContext('2d');\n";
  
  html += "  function drawChart(history) {\n";
  html += "    ctx.clearRect(0, 0, canvas.width, canvas.height);\n";
  
  // Draw Grid Lines
  html += "    ctx.strokeStyle = '#334155';\n";
  html += "    ctx.lineWidth = 1;\n";
  html += "    for(let y = 50; y < 200; y += 50) {\n";
  html += "      ctx.beginPath();\n";
  html += "      ctx.moveTo(0, y);\n";
  html += "      ctx.lineTo(canvas.width, y);\n";
  html += "      ctx.stroke();\n";
  html += "    }\n";
  
  // Draw Target Centerline (15 cm)
  html += "    ctx.strokeStyle = '#ef4444';\n";
  html += "    ctx.setLineDash([5, 5]);\n";
  html += "    ctx.beginPath();\n";
  html += "    const targetY = 200 - (15 * 4); // Scale 1cm = 4px\n";
  html += "    ctx.moveTo(0, targetY);\n";
  html += "    ctx.lineTo(canvas.width, targetY);\n";
  html += "    ctx.stroke();\n";
  html += "    ctx.setLineDash([]); // Reset line style\n";
  
  // Draw Data Path line
  html += "    ctx.strokeStyle = '#38bdf8';\n";
  html += "    ctx.lineWidth = 3;\n";
  html += "    ctx.beginPath();\n";
  
  html += "    const step = canvas.width / (history.length - 1);\n";
  html += "    history.forEach((val, i) => {\n";
  html += "      const x = i * step;\n";
  html += "      const y = 200 - (val * 4); // Scale factor\n";
  html += "      if (i === 0) ctx.moveTo(x, y);\n";
  html += "      else ctx.lineTo(x, y);\n";
  html += "    });\n";
  html += "    ctx.stroke();\n";
  html += "  }\n";
  
  html += "  function updateHUD() {\n";
  html += "    fetch('/api/telemetry')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('distVal').innerText = data.distance.toFixed(1) + ' cm';\n";
  
  html += "        const bdgEl = document.getElementById('actionDisplay');\n";
  html += "        bdgEl.innerText = data.action;\n";
  html += "        if (data.action === 'FORWARD') {\n";
  html += "          bdgEl.className = 'status-badge';\n";
  html += "        } else {\n";
  html += "          bdgEl.className = 'status-badge adjusting';\n";
  html += "        }\n";
  
  html += "        drawChart(data.history);\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateHUD();\n";
  html += "    setInterval(updateHUD, 200); // Fast poll (5 times a second)\n";
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
  
  // Fill history buffer with default 15cm readings
  for (int i = 0; i < HISTORY_SIZE; i++) {
    distanceHistory[i] = 15.0;
  }
  
  Serial.println("\nESP32 Wall Follower starting...");
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
}

void loop() {
  server.handleClient();
  
  // 1. Measure side wall distance (right side-mounted)
  currentDistanceCm = readDistanceCm();
  addHistoryReading(currentDistanceCm);
  
  // 2. Closed-Loop Wall Following Control Loop
  // Target: 15 cm. Deadband range: 12 cm to 18 cm.
  if (currentDistanceCm < 12.0) {
    // Too close to right wall -> steer left (away)
    steerLeft();
  } 
  else if (currentDistanceCm > 18.0) {
    // Too far from right wall -> steer right (toward)
    steerRight();
  } 
  else {
    // In range -> Proceed forward
    moveForward();
  }
  
  delay(20); // Small interval delay
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **HC-SR04 Sensor**, **L298N Driver**, and **2 DC Motors** onto the canvas.
2. Wire TRIG/ECHO to **GPIO12/GPIO14**, and IN1–IN4 to **GPIO27, 26, 25, and 33**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. In normal state, set the simulated ultrasonic sensor distance to 15 cm. Verify that the motors spin forward (green clockwise/counterclockwise).
6. Slide the simulated distance below 10 cm (too close). Verify that the left motor reverses and the right motor spins forward (steering left), and the line chart updates dynamically.
7. Slide the distance to 25 cm (too far). Verify that the robot steers right.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
```

Browser JSON Console (`/api/telemetry`):
```json
{"distance":11.4,"action":"STEER LEFT","history":[15.0,15.0,14.2,13.1,11.4]}
```

## Expected Canvas Behavior
* Adjusting the simulated ultrasonic distance shifts motor rotation directions to keep the robot on path.
* The webpage graph displays a scrolling track of the distance history.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `addHistoryReading(...)` | Appends the current distance value to the circular history buffer. |
| `currentDistanceCm < 12.0` | Threshold comparison checking if the robot is too close to the wall. |
| `currentDistanceCm > 18.0` | Threshold comparison checking if the robot is too far from the wall. |
| `ctx.lineTo(x, y)` | Draws line segments connecting data points on the HTML5 canvas. |

## Hardware & Safety Concept: Sensor Glitch Filtering and Proportional steering
* **Sensor Glitch Filtering**: Ultrasonic sensors can suffer from acoustic reflections, returning random spikes (e.g. reading 400 cm when the wall is at 15 cm). If not filtered, this causes the robot to steer violently. Implement a running median filter or ignore readings that differ from the previous value by more than 50% to smooth control.
* **Proportional steering**: Instead of sudden left/right digital steer adjustments, calculate the error (distance - 15 cm) and adjust motor speeds proportionally using PWM. This provides smoother tracking along the wall.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a real-time line graph of the wall distance locally.
2. **Reverse buzzer**: Add a buzzer (GPIO 15) to beep when reversing.
3. **SPIFFS integration**: Save wall tracking logs to a file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The graph line goes off-screen | Scaling factor wrong | Adjust the canvas y-coordinate scaling in the script (e.g. `200 - (val * 4)`) |
| Robot steers toward wall when too close | Motor wiring reversed | Swap the direction pins IN1/IN2 or IN3/IN4 in code |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [50 - Potentiometer servo angle controller](../beginner/50-potentiometer-servo-angle-controller-bar-display.md)
- [124 - ESP32 Obstacle Avoidance Robot with Web HUD](124-esp32-obstacle-avoidance-robot-with-web-hud-telemetry.md)
- [130 - ESP32 Light Seeking Robotic Bug Web Status](130-esp32-light-seeking-robotic-bug-web-status.md) (Next project)
