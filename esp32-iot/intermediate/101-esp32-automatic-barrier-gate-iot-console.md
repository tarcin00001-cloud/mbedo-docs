# 101 - Automatic Barrier Gate IoT Console (IR sensor + Servo + LCD + Web logs)

Build an automated security barrier gate system on the ESP32 that detects obstacles using an IR sensor on GPIO 14, opens a servo gate on GPIO 13, displays status information on a 16x2 I2C LCD, and hosts a web dashboard with real-time logs of gate events.

## Goal
Learn how to read digital IR obstacle sensors, drive servo motors using `ESP32Servo`, write non-blocking state delays, print diagnostics to I2C LCDs, and maintain dynamic event logs in web pages.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An IR obstacle sensor is on GPIO 14, a servo on GPIO 13, and a 16x2 I2C LCD on I2C (GPIO 21/22). When a vehicle is detected, the servo rotates 90 degrees (gate open), the LCD displays "GATE: OPEN", and an entry is logged. When the vehicle leaves, the gate closes after a 3-second delay, and the event is logged. A web page displays the gate state and a table of recent events.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| IR Obstacle Sensor | `button` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use a pushbutton configured as active-LOW (internal pull-up) to simulate the IR obstacle sensor output.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| IR Sensor | OUT (Signal) | GPIO14 | Yellow | Obstacle detector input |
| Servo Motor | PWM (Signal) | GPIO13 | Orange | Gate barrier actuator |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the servo and LCD from the 5V Vin rail.

## Code
```cpp
// Automatic Barrier Gate IoT Console (IR triggering + Non-blocking Servo gate + Web logs)
#include <WiFi.h>
#include <WebServer.h>
#include <ESP32Servo.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int IR_PIN = 14;
const int SERVO_PIN = 13;

Servo gateServo;
WebServer server(80);

// Gate States
enum GateState { CLOSED, OPENING, OPEN, CLOSING };
GateState currentState = CLOSED;

unsigned long gateTimer = 0;
const unsigned long GATE_HOLD_TIME_MS = 3000; // Hold gate open for 3 seconds

// Rolling Event Log (Stores last 5 events)
String eventLogs[5] = {"", "", "", "", ""};
int logCount = 0;

void addLog(String message) {
  // Shift logs
  for (int i = 4; i > 0; i--) {
    eventLogs[i] = eventLogs[i - 1];
  }
  
  // Format entry
  unsigned long timestampS = millis() / 1000;
  eventLogs[0] = "[" + String(timestampS) + "s] " + message;
  
  if (logCount < 5) logCount++;
  Serial.println("[LOG] " + eventLogs[0]);
}

// Get gate state label
String getGateStateString() {
  switch (currentState) {
    case CLOSED:  return "CLOSED";
    case OPENING: return "OPENING";
    case OPEN:    return "OPEN";
    case CLOSING: return "CLOSING";
  }
  return "UNKNOWN";
}

void updateLCDDisplay() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("GATE: ");
  lcd.print(getGateStateString());
  
  lcd.setCursor(0, 1);
  if (digitalRead(IR_PIN) == LOW) {
    lcd.print("Vehicle Detected");
  } else {
    lcd.print("Clear");
  }
}

// HTTP API endpoint returning JSON data
void handleGetGateData() {
  String json = "{\"state\":\"" + getGateStateString() + "\",\"logs\":[";
  for (int i = 0; i < 5; i++) {
    json += "\"" + eventLogs[i] + "\"";
    if (i < 4) json += ",";
  }
  json += "]}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Barrier Gate Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .status-badge { display: inline-block; padding: 10px 20px; border-radius: 20px; font-weight: bold; font-size: 16px; text-transform: uppercase; margin: 20px 0; }\n";
  html += "  .closed { background-color: #991b1b; color: #fee2e2; }\n";
  html += "  .open { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .transition { background-color: #92400e; color: #fef3c7; }\n";
  html += "  table { width: 100%; border-collapse: collapse; margin-top: 25px; text-align: left; font-size: 13px; }\n";
  html += "  th { color: #64748b; border-bottom: 2px solid #334155; padding-bottom: 8px; }\n";
  html += "  td { padding: 8px 0; border-bottom: 1px solid #1e293b; color: #94a3b8; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Security Barrier Gate</h1>\n";
  html += "  <div id=\"gateBadge\" class=\"status-badge closed\">CLOSED</div>\n";
  
  html += "  <h3>Recent Events</h3>\n";
  html += "  <table>\n";
  html += "    <thead><tr><th>Activity Logs</th></tr></thead>\n";
  html += "    <tbody id=\"logTable\"></tbody>\n";
  html += "  </table>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 1 second | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateGateStatus() {\n";
  html += "    fetch('/api/gate')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        const bdgEl = document.getElementById('gateBadge');\n";
  html += "        bdgEl.innerText = data.state;\n";
  
  html += "        if (data.state === 'CLOSED') {\n";
  html += "          bdgEl.className = 'status-badge closed';\n";
  html += "        } else if (data.state === 'OPEN') {\n";
  html += "          bdgEl.className = 'status-badge open';\n";
  html += "        } else {\n";
  html += "          bdgEl.className = 'status-badge transition';\n";
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
  html += "    updateGateStatus();\n";
  html += "    setInterval(updateGateStatus, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(IR_PIN, INPUT_PULLUP);
  
  // Initialize LCD screen
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Barrier Setup");
  lcd.setCursor(0, 1);
  lcd.print("Connecting WiFi");
  
  gateServo.attach(SERVO_PIN);
  gateServo.write(0); // Initialize gate CLOSED
  
  Serial.println("\nESP32 Barrier Gate Station Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/gate", handleGetGateData);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  addLog("System Initialized");
  updateLCDDisplay();
}

void loop() {
  server.handleClient();
  
  // Read IR Obstacle Sensor (LOW when blocked, HIGH when clear)
  bool carDetected = (digitalRead(IR_PIN) == LOW);
  
  // State Machine control
  switch (currentState) {
    case CLOSED:
      if (carDetected) {
        addLog("Vehicle Detected. Opening Gate...");
        gateServo.write(90); // Open gate
        currentState = OPEN;
        updateLCDDisplay();
      }
      break;
      
    case OPEN:
      if (carDetected) {
        // Reset timer if car is still standing in front of the gate
        gateTimer = millis();
      } else {
        // Start countdown to close gate once car leaves
        if (millis() - gateTimer >= GATE_HOLD_TIME_MS) {
          addLog("Clear. Closing Gate...");
          gateServo.write(0); // Close gate
          currentState = CLOSED;
          updateLCDDisplay();
        }
      }
      break;
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Pushbutton** (configured as active-LOW to simulate the IR sensor), **Servo**, and **16x2 I2C LCD** onto the canvas.
2. Wire Button to **GPIO14**, Servo to **GPIO13**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Click and hold the button widget. Verify that the servo opens to 90 degrees, the LCD shows "GATE: OPEN", and the web dashboard logs the event.
6. Release the button. Verify that the gate closes after 3 seconds, and the log table updates automatically.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[LOG] [0s] System Initialized
[LOG] [5s] Vehicle Detected. Opening Gate...
[LOG] [10s] Clear. Closing Gate...
```

LCD Display:
```
GATE: OPEN
Vehicle Detected
```

## Expected Canvas Behavior
* Pressing the simulated IR sensor button immediately opens the servo gate widget.
* Releasing the button triggers a 3-second hold countdown before the servo returns to 0° (closed).

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `carDetected = (digitalRead(IR_PIN) == LOW)` | Reads the active-LOW IR obstacle detection signal. |
| `gateServo.write(90)` | Rotates the servo shaft to position the gate arm upwards. |
| `millis() - gateTimer >= GATE_HOLD_TIME_MS` | Enforces a non-blocking 3-second hold period before closing the gate. |
| `eventLogs[i] = eventLogs[i - 1]` | Shifts array elements to maintain a rolling logs list. |

## Hardware & Safety Concept: Vehicle Presence Detection and Safety Loops
Automatic barrier gates present a crushing hazard to vehicles and pedestrians. To protect them:
1. **Safety Interlock**: If the gate is closing and a new car is detected by the IR sensor, the gate must stop and open immediately.
2. **Dual-Sensor Loops**: Production systems combine IR beams with magnetic loop detectors in the ground to detect metal vehicle chassis, preventing the gate from closing while a vehicle is parked underneath.

## Try This! (Challenges)
1. **OLED Status Display**: Upgrade the screen to an SSD1306 OLED (Project 60) and draw a barrier gate icon.
2. **Access Card Control**: Add an RFID reader (Project 109) and require a valid card to trigger gate opening.
3. **Emergency Override Button**: Add a second button (GPIO 12) to force the gate to remain open.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo does not move | Power rails missing | Confirm that the servo is powered from the 5V Vin rail, not the 3.3V rail |
| The gate closes immediately | Timer reset missing | Ensure `gateTimer` is updated continuously while the IR sensor remains blocked |
| LCD does not print text | I2C address mismatch | Verify the I2C address in `LiquidCrystal_I2C` matches your LCD module (usually `0x27`) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [56 - ESP32 Web Page Servo angle controller](56-esp32-web-page-servo-angle-controller-oled-display-feedback.md)
- [100 - ESP32 Automatic Smart Fan IoT Console](100-esp32-automatic-smart-fan-iot-console.md)
- [102 - ESP32 Water Level Control Station IoT](102-esp32-water-level-control-station-iot-water-level-relay-buzzer-web-status.md) (Next project)
