# 165 - Smart Door Lock with Web Access Override logs

Build an internet-connected smart lock system on the ESP32 that controls a door latch servo on GPIO 2, accepts physical unlock button triggers on GPIO 14, logs all access events to a local file in SPIFFS flash memory, and hosts a web dashboard with remote lock controls and historical log viewers.

## Goal
Learn how to mount and read/write the SPIFFS (Serial Peripheral Interface Flash File System) local flash memory, handle debounced physical inputs, steer servo motors, and serve web administration panels.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A door lock servo is on GPIO 2, a physical exit button on GPIO 14, and an status LED on GPIO 12. Pressing the physical button or clicking the unlock button on the hosted web page unlocks the door (rotating the servo to 90°). Every unlock event is written to `/lock_log.txt` in the ESP32's flash memory. Navigating to the ESP32's IP address displays a webpage displaying the current state and a log of past access events.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| Pushbutton Switch | `pushbutton` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Servo Motor | PWM (Signal) | GPIO2 | Yellow | Latch controller |
| Pushbutton | Pin 1 / Pin 2 | GPIO14 / GND | Blue / Black | Local exit switch |
| LED | Anode (+) | GPIO12 | Red | Status indicator |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Configure the pushbutton pin as `INPUT_PULLUP` to eliminate the need for an external resistor.

## Code
```cpp
// Smart Door Lock with Web Access (SPIFFS flash data logger + Servo Latch + Web Controls)
#include <WiFi.h>
#include <WebServer.h>
#include <SPIFFS.h>
#include <ESP32Servo.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int SERVO_PIN = 2;
const int BUTTON_PIN = 14;
const int LED_PIN = 12;

Servo lockServo;
WebServer server(80);

// Lock state variables
bool isLocked = true;
unsigned long unlockTime = 0;
const unsigned long AUTO_RELOCK_DELAY_MS = 4000; // Auto-relock after 4 seconds

// SPIFFS logs path
const char* logFilepath = "/lock_log.txt";

// Write access log events to SPIFFS flash memory
void appendLog(String eventMsg) {
  File logFile = SPIFFS.open(logFilepath, FILE_APPEND);
  if (logFile) {
    unsigned long timestampS = millis() / 1000;
    logFile.printf("[%lu s] %s\n", timestampS, eventMsg.c_str());
    logFile.close();
    Serial.println("[SPIFFS LOG] " + eventMsg);
  } else {
    Serial.println("[SPIFFS Error] Failed to open log file for appending!");
  }
}

// Read log history from SPIFFS
String readLogs() {
  File logFile = SPIFFS.open(logFilepath, FILE_READ);
  if (!logFile) return "No logs available.";
  
  String fileContent = "";
  // Read last 600 characters to prevent buffer overflows
  if (logFile.size() > 600) {
    logFile.seek(logFile.size() - 600);
  }
  
  while (logFile.available()) {
    fileContent += (char)logFile.read();
  }
  logFile.close();
  return fileContent;
}

// Perform unlock operation
void performUnlock(String triggerSource) {
  if (isLocked) {
    isLocked = false;
    unlockTime = millis();
    lockServo.write(90); // Rotate servo to unlocked position (90 degrees)
    digitalWrite(LED_PIN, HIGH); // Turn ON LED indicator
    appendLog("Unlocked via " + triggerSource);
  }
}

// Perform lock operation
void performLock() {
  if (!isLocked) {
    isLocked = true;
    lockServo.write(0); // Rotate servo back to locked position (0 degrees)
    digitalWrite(LED_PIN, LOW); // Turn OFF LED
    appendLog("Door Relocked");
  }
}

// HTTP API endpoint returning JSON data
void handleGetStatus() {
  String json = "{\"locked\":" + String(isLocked ? "true" : "false") + 
                 ",\"logs\":\"" + readLogs() + "\"}";
  // Replace newlines with HTML line breaks in response string
  json.replace("\n", "\\n");
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to trigger locks
void handlePostLockControl() {
  if (server.hasArg("action")) {
    String action = server.arg("action");
    if (action == "unlock") {
      performUnlock("Web Dashboard");
    } else if (action == "lock") {
      performLock();
    }
    server.send(200, "text/plain", "OK");
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Smart Lock Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .status-badge { display: inline-block; padding: 10px 20px; border-radius: 20px; font-weight: bold; font-size: 15px; text-transform: uppercase; margin: 20px 0; background-color: #7f1d1d; color: #fca5a5; }\n";
  html += "  .status-badge.unlocked { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 14px; font-size: 16px; font-weight: bold; color: white; background-color: #38bdf8; border: none; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; margin-top: 10px; }\n";
  html += "  .btn-unlock { background-color: #10b981; }\n";
  html += "  .btn-unlock:hover { background-color: #059669; }\n";
  html += "  .btn-lock { background-color: #ef4444; }\n";
  html += "  .btn-lock:hover { background-color: #dc2626; }\n";
  html += "  .log-box { background-color: #0f172a; border-radius: 8px; border: 1px solid #334155; padding: 15px; margin-top: 25px; text-align: left; font-family: monospace; font-size: 12px; height: 120px; overflow-y: auto; white-space: pre-wrap; color: #94a3b8; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Smart Door Lock</h1>\n";
  html += "  <div id=\"statusBadge\" class=\"status-badge\">LOCKED</div>\n";
  
  html += "  <button class=\"btn btn-unlock\" onclick=\"controlLock('unlock')\">Remote Unlock</button>\n";
  html += "  <button class=\"btn btn-lock\" onclick=\"controlLock('lock')\">Remote Lock</button>\n";
  
  html += "  <h3>Access History (SPIFFS)</h3>\n";
  html += "  <div id=\"logBox\" class=\"log-box\">Loading logs...</div>\n";
  
  html += "  <p class=\"footer\">Local exit button: GPIO 14 | Auto-relock: 4s</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateHUD() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        const bdgEl = document.getElementById('statusBadge');\n";
  html += "        if (data.locked) {\n";
  html += "          bdgEl.innerText = 'LOCKED';\n";
  html += "          bdgEl.className = 'status-badge';\n";
  html += "        } else {\n";
  html += "          bdgEl.innerText = 'UNLOCKED';\n";
  html += "          bdgEl.className = 'status-badge unlocked';\n";
  html += "        }\n";
  html += "        document.getElementById('logBox').innerText = data.logs;\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  function controlLock(actionVal) {\n";
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('action', actionVal);\n";
  html += "    fetch('/api/control', { method: 'POST', body: body })\n";
  html += "      .then(() => updateHUD());\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateHUD();\n";
  html += "    setInterval(updateHUD, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Mount SPIFFS
  Serial.println("\nMounting SPIFFS filesystem...");
  if (!SPIFFS.begin(true)) {
    Serial.println("[SPIFFS Error] Mount failed! Formatting...");
  } else {
    Serial.println("[SPIFFS] Mounted successfully.");
  }
  
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  
  lockServo.attach(SERVO_PIN);
  lockServo.write(0); // CLOSED/LOCKED position
  
  Serial.println("\nESP32 Smart Lock starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/status", handleGetStatus);
  server.on("/api/control", HTTP_POST, handlePostLockControl);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  appendLog("Lock System Booted");
}

void loop() {
  server.handleClient();
  
  // 1. Read Physical debounced push button (Active-LOW Exit button)
  static unsigned long lastBtnTime = 0;
  if (digitalRead(BUTTON_PIN) == LOW) {
    if (millis() - lastBtnTime > 250) { // Simple debounce
      lastBtnTime = millis();
      performUnlock("Physical Exit Button");
    }
  }
  
  // 2. Automated Auto-Relock routine
  if (!isLocked && (millis() - unlockTime >= AUTO_RELOCK_DELAY_MS)) {
    performLock();
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Servo Motor**, **Pushbutton**, and **LED** onto the canvas.
2. Wire Servo to **GPIO2**, Pushbutton to **GPIO14**, and LED to **GPIO12**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Click the simulated Pushbutton widget. Verify that the servo rotates to 90 degrees, the LED turns ON, and the status display updates to `UNLOCKED`.
6. Wait 4 seconds. Verify that the servo rotates back to 0 degrees (locked) automatically.
7. Click "Remote Unlock" on the webpage. Verify that the lock opens and the action is appended to the SPIFFS history box.

## Expected Output
Serial Monitor:
```
[SPIFFS] Mounted successfully.
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[SPIFFS] Saved entry -> Unlocked via Physical Exit Button
[SPIFFS] Saved entry -> Door Relocked
```

## Expected Canvas Behavior
* Pressing the canvas pushbutton rotates the servo latch widget and turns the LED ON.
* Webpage buttons open and close the physical lock servo.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `SPIFFS.begin(true)` | Mounts the SPIFFS filesystem (formatting it if mounting fails). |
| `SPIFFS.open(..., FILE_APPEND)` | Appends access events to the text log file in flash. |
| `lockServo.write(90)` | Rotates the servo to clear the physical door strike plate. |
| `millis() - unlockTime >= AUTO_RELOCK_DELAY_MS` | Auto-lock timeout comparison check. |

## Hardware & Safety Concept: Fire Safety Fail-safe Systems
* **Fire Safety (Fail-Safe)**: In commercial buildings, electronic locks must be designed to fail-safe. If power is lost or the fire alarm triggers, the lock must open automatically to allow occupants to escape. Mechanical locks are typically held closed by electromagnets that release when power is cut. For servo-driven deadbolts, always include a mechanical handle override on the inside of the door.
* **Flash Wear Mitigation**: SPIFFS write cycles are limited (typically 10,000 to 100,000 cycles). Avoid writing logs to flash too frequently. Group events or keep log entries concise to extend flash memory life.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a locked padlock icon.
2. **Alert buzzer**: Add a buzzer (GPIO 15) to sound a warning chime when the door is left open for too long.
3. **Delete Logs Button**: Add an API route to format SPIFFS and clear log history from the dashboard.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| SPIFFS Mount Failed | File system corruption | Set the `begin(true)` flag to force formatting the flash memory partition on startup |
| Servo does not rotate | Power supply weak | Servo motors draw high surge currents. Powering the servo directly from the ESP32's 3.3V rail will cause the ESP32 to brown out and reset. Always connect the servo power line to the 5V rail |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [62 - ESP32 SPIFFS flash memory data logger](../intermediate/62-esp32-spiffs-flash-memory-data-logger.md)
- [143 - Firebase Authentication Access Controller](143-firebase-authentication-access-controller.md)
- [158 - Keypad RFID Multi-factor Lock](158-esp32-keypad-rfid-multi-factor-lock-iot-with-remote-open-url-override.md)
- [166 - Smart Fan with Web Temperature Threshold adjustment](166-esp32-smart-fan-with-web-temperature-threshold-adjustment.md) (Next project)
