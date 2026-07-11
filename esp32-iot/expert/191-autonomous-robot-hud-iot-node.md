# 191 - Autonomous Robot HUD IoT Node (Core 0 drives, Core 1 hosts WebSocket chart)

Build a multi-core autonomous robot controller on the ESP32 that runs a hardware-timed driving loop on Core 0 (HC-SR04 distance tracking and L298N motor drivers) and serves a low-latency WebSocket diagnostic HUD on Core 1 using FreeRTOS task pinning.

## Goal
Learn how to pin FreeRTOS tasks to specific ESP32 processor cores, implement thread-safe global data exchanges using mutexes, stream high-frequency WebSocket telemetries, and render live canvas-based scrolling time-series graphs.

## What You Will Build
An ESP32 DevKitC acts as a dual-core robot brain. Core 0 handles autonomous obstacle avoidance: reading an HC-SR04 sensor (GPIO 14, 15) and steering DC motors using an L298N driver. Simultaneously, Core 1 connects to WiFi, runs an HTTP server on port 80, and broadcasts telemetry packets (current distance and speed) over WebSockets on port 81 to a dashboard displaying a scrolling canvas graph.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| L298N Motor Driver + 2x DC Motors | `l298n` / `motor` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 | Trigger | GPIO14 | Yellow | Trigger pulse input |
| HC-SR04 | Echo | GPIO15 | Green | Echo pulse output |
| L298N Driver | IN1 / IN2 | GPIO25 / GPIO26 | Blue / Violet | Left Motor PWM channels |
| L298N Driver | IN3 / IN4 | GPIO32 / GPIO33 | Orange / Yellow | Right Motor PWM channels |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Secure separate batteries for the motors and the ESP32 to prevent electrical noise from resetting the microcontroller.

## Code
```cpp
// Multi-Core Robot Controller (FreeRTOS Core Pinning + Mutual Exclusions + WebSockets + Canvas Graph)
#include <WiFi.h>
#include <WebServer.h>
#include <WebSocketsServer.h>
#include <ArduinoJson.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// HC-SR04 Pins
const int TRIG_PIN = 14;
const int ECHO_PIN = 15;

// Motor Driver Pins (Proportional LEDC PWM)
const int IN1 = 25;
const int IN2 = 26;
const int IN3 = 32;
const int IN4 = 33;

WebServer server(80);
WebSocketsServer webSocket = WebSocketsServer(81);

// Thread-safe global robot data structure
struct RobotState {
  float distanceCm;
  int leftSpeed;
  int rightSpeed;
  char direction[16];
};
RobotState robotState = {100.0, 0, 0, "STOP"};

// FreeRTOS Mutex to prevent data race conditions between Core 0 and Core 1
SemaphoreHandle_t stateMutex;

// Forward drive control
void driveForward(int speed) {
  analogWrite(IN1, speed);
  analogWrite(IN2, 0);
  analogWrite(IN3, speed);
  analogWrite(IN4, 0);
}

// Right pivot steering
void turnRight(int speed) {
  analogWrite(IN1, speed);
  analogWrite(IN2, 0);
  analogWrite(IN3, 0);
  analogWrite(IN4, speed);
}

// Full stop
void stopMotors() {
  analogWrite(IN1, 0);
  analogWrite(IN2, 0);
  analogWrite(IN3, 0);
  analogWrite(IN4, 0);
}

// Core 0 Task: Handles Time-Critical Sensor Reading and Motor Control Loops
void Core0DrivingTask(void * pvParameters) {
  Serial.print("[FreeRTOS] Core 0 Task started on core: ");
  Serial.println(xPortGetCoreID());
  
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  
  while (true) {
    // 1. Measure obstacle distance using HC-SR04
    digitalWrite(TRIG_PIN, LOW);
    delayMicroseconds(2);
    digitalWrite(TRIG_PIN, HIGH);
    delayMicroseconds(10);
    digitalWrite(TRIG_PIN, LOW);
    
    long duration = pulseIn(ECHO_PIN, HIGH, 30000); // 30 ms timeout
    float distance = (duration > 0) ? (duration * 0.0343 / 2.0) : 400.0;
    
    // 2. Obstacle Avoidance Decision Tree
    int lSpeed = 0, rSpeed = 0;
    String dir = "STOP";
    
    if (distance < 20.0) {
      // Obstacle ahead! Stop, turn right to clear
      stopMotors();
      delay(200);
      turnRight(180);
      lSpeed = 180; rSpeed = -180;
      dir = "TURN RIGHT";
      delay(500); // Turn block
    } else {
      // Path clear. Move forward.
      driveForward(200);
      lSpeed = 200; rSpeed = 200;
      dir = "FORWARD";
    }
    
    // 3. Update global state using Mutex Lock
    if (xSemaphoreTake(stateMutex, pdMS_TO_TICKS(10)) == pdTRUE) {
      robotState.distanceCm = distance;
      robotState.leftSpeed = lSpeed;
      robotState.rightSpeed = rSpeed;
      strcpy(robotState.direction, dir.c_str());
      xSemaphoreGive(stateMutex);
    }
    
    vTaskDelay(pdMS_TO_TICKS(50)); // Run control loop at 20 Hz
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Robot Telemetry HUD</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 600px; width: 100%; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; text-align: center; margin-top: 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 20px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 24px; font-weight: bold; margin-top: 5px; color: #10b981; font-family: monospace; }\n";
  html += "  canvas { width: 100%; height: 180px; background-color: #0f172a; border-radius: 8px; border: 1px solid #334155; margin-top: 20px; }\n";
  html += "  .footer { text-align: center; margin-top: 25px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Multi-Core Robot HUD</h1>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Obstacle Distance</div><div class=\"metric-val\" id=\"distVal\">0.0 cm</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Drive State</div><div class=\"metric-val\" id=\"dirVal\" style='color:#38bdf8;'>STOP</div></div>\n";
  html += "  </div>\n";
  
  html += "  <canvas id=\"chart\"></canvas>\n";
  
  html += "  <p class=\"footer\">Core 0: Driving Engine (20Hz) | Core 1: WebSockets Server</p>\n";
  html += "</div>\n";
  
  // Real-time Canvas Graphing Script
  html += "<script>\n";
  html += "  const canvas = document.getElementById('chart');\n";
  html += "  const ctx = canvas.getContext('2d');\n";
  html += "  const history = new Array(100).fill(100); // Circular history buffer\n";
  
  html += "  function drawChart() {\n";
  html += "    ctx.clearRect(0, 0, canvas.width, canvas.height);\n";
  html += "    ctx.beginPath();\n";
  html += "    ctx.strokeStyle = '#10b981';\n";
  html += "    ctx.lineWidth = 2;\n";
  
  html += "    const step = canvas.width / (history.length - 1);\n";
  html += "    for (let i = 0; i < history.length; i++) {\n";
  // Map 0-200cm distance to canvas height
  html += "      const y = canvas.height - (history[i] / 200.0) * canvas.height;\n";
  html += "      if (i === 0) ctx.moveTo(0, y); else ctx.lineTo(i * step, y);\n";
  html += "    }\n";
  html += "    ctx.stroke();\n";
  html += "  }\n";
  
  // WebSocket Connection
  html += "  const ws = new WebSocket('ws://' + window.location.hostname + ':81/');\n";
  html += "  ws.onmessage = function(event) {\n";
  html += "    const data = JSON.parse(event.data);\n";
  html += "    document.getElementById('distVal').innerText = data.dist.toFixed(1) + ' cm';\n";
  html += "    document.getElementById('dirVal').innerText = data.dir;\n";
  
  html += "    history.push(data.dist);\n";
  html += "    history.shift(); // Maintain buffer length\n";
  html += "    drawChart();\n";
  html += "  };\n";
  
  html += "  window.onload = function() {\n";
  html += "    canvas.width = canvas.clientWidth;\n";
  html += "    canvas.height = canvas.clientHeight;\n";
  html += "    drawChart();\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Initialize Mutex
  stateMutex = xSemaphoreCreateMutex();
  
  Serial.println("\nESP32 Multi-Core Setup starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  Serial.print("Local IP Address: ");
  Serial.println(WiFi.localIP());
  
  // Start WebServer & WebSocketsServer
  server.on("/", handleRoot);
  server.begin();
  webSocket.begin();
  
  // 3. Pin Autonomous Driving Task to Core 0
  // Parameters: TaskFunc, TaskName, StackSize, ParamPtr, Priority, TaskHandle, CoreID (0)
  xTaskCreatePinnedToCore(
    Core0DrivingTask,
    "DrivingEngine",
    4096,
    NULL,
    2,
    NULL,
    0
  );
  
  Serial.println("System initialized. Core 1 hosting Web engine, Core 0 driving motors.");
}

void loop() {
  // Core 1 runs the standard Arduino loop functions (WebServer + WebSockets processing)
  server.handleClient();
  webSocket.loop();
  
  // Broadcast telemetry updates over WebSockets once every 200 ms
  static unsigned long lastBroadcast = 0;
  if (millis() - lastBroadcast > 200) {
    lastBroadcast = millis();
    
    float dist = 0.0;
    String dir = "";
    
    // Copy states using Mutex Lock
    if (xSemaphoreTake(stateMutex, pdMS_TO_TICKS(10)) == pdTRUE) {
      dist = robotState.distanceCm;
      dir = String(robotState.direction);
      xSemaphoreGive(stateMutex);
    }
    
    // Compile JSON string and broadcast to all connected WebSocket clients
    String jsonPayload = "{\"dist\":" + String(dist, 1) + ",\"dir\":\"" + dir + "\"}";
    webSocket.broadcastTXT(jsonPayload);
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **HC-SR04 Sensor**, and **L298N Motor Driver** onto the canvas.
2. Wire the Trigger and Echo pins to **GPIO14** and **GPIO15**. Wire L298N inputs to the specified pins.
3. Paste the code and click **Run**.
4. Set the simulated HC-SR04 distance slider to 100 cm. Verify that the DC motor widgets start spinning forward.
5. Drag the distance slider down to 10 cm. Verify that the motors stop, change directions (steering right), and then resume spinning forward.
6. Open your web browser and navigate to `http://10.10.0.3/` to view the diagnostic canvas chart plotting the distance readings in real-time.

## Expected Output
Serial Monitor:
```
WiFi Connected.
Local IP Address: 10.10.0.3
[FreeRTOS] Core 0 Task started on core: 0
System initialized. Core 1 hosting Web engine, Core 0 driving motors.
```

## Expected Canvas Behavior
* Driving the robot triggers motor widget movements on Core 0. Navigating to the page opens a WebSocket channel that plots time-series graphs without delay.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `xTaskCreatePinnedToCore(...)` | FreeRTOS function that binds a function loop to run exclusively on Core 0. |
| `xSemaphoreTake(...)` | Requests exclusive access to lock the mutex before writing/reading data. |
| `webSocket.broadcastTXT(...)` | Transmits the JSON string to all clients connected to port 81. |

## Hardware & Safety Concept: Mutex Deadlocks and Motor Electrical Isolation
* **Mutex Deadlocks**: Mutexes prevent Core 0 and Core 1 from writing to the same memory address simultaneously (which causes data corruption). However, if Core 0 locks the mutex and gets stuck, Core 1 will block indefinitely waiting for release. Always specify a timeout (e.g. `pdMS_TO_TICKS(10)`) when taking a mutex instead of waiting forever.
* **Motor Electrical Isolation**: DC motors generate large back-EMF spikes when starting or changing directions. These voltage spikes can travel back through the power rail and cause the ESP32 to brown out or reset. Always power the motor driver board from a separate battery pack, connecting only the GND pins together.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display "Core 0: Driving | Core 1: WS".
2. **Dynamic Speed Adjustment**: Add sliders to the web interface to change the motor speeds dynamically using WebSocket commands.
3. **Compass Balancer**: Add an MPU-6050 accelerometer (Project 155) to display the robot's real-time pitch/roll orientations on the web dashboard.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| ESP32 crashes repeatedly on boot | Stack overflow | FreeRTOS tasks require allocated stack memory. If the task function executes complex libraries or strings, increase the stack parameter from `4096` to `8192` |
| Web page loads but shows no data | WebSocket blocked | Ensure port 81 is not blocked by a local firewall or proxy server, and check that the WebSocket client URL is correct |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [112 - Obstacle Avoider Robot](../intermediate/112-obstacle-avoider.md)
- [152 - Web Page Motor PID Speed Graph](152-esp32-web-page-motor-pid-speed-graph.md)
- [192 - Asynchronous Multi-Sensor Climate Node](192-asynchronous-multi-sensor-climate-node.md) (Next project)
