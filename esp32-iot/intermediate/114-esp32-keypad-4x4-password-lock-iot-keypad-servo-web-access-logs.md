# 114 - Keypad 4x4 Password Lock IoT (Keypad + Servo + Web access logs)

Build an automated security lock system on the ESP32 that reads a 4x4 matrix keypad, drives a lock servo on GPIO 13, and hosts a web dashboard displaying lock states and access logs.

## Goal
Learn how to scan matrix keypads using the Keypad library, drive servo motors using `ESP32Servo`, implement non-blocking lock delays, serve web control panels, and process remote unlock requests.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A 4x4 matrix keypad is connected to GPIO pins, and a lock servo is on GPIO 13. Entering the correct passcode (`1234`) on the keypad rotates the servo to 90 degrees (unlocked) for 5 seconds and logs the access event. Entering an incorrect passcode logs a warning. A web page shows the lock status, a table of recent events, and an unlock button.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| 4x4 Keypad | Rows 1 - 4 | GPIO12, 14, 27, 26 | Yellow | Row scan lines |
| 4x4 Keypad | Cols 1 - 4 | GPIO25, 33, 32, 35 | Green | Column scan lines |
| Servo Motor | PWM (Signal) | GPIO13 | Orange | Lock bolt actuator |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the servo from the 5V Vin rail.

## Code
```cpp
// Keypad 4x4 Password Lock IoT (Keypad entry check + Servo lock + Web logs)
#include <WiFi.h>
#include <WebServer.h>
#include <ESP32Servo.h>
#include <Keypad.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int SERVO_PIN = 13;
Servo lockServo;

WebServer server(80);

// Keypad configuration
const byte ROWS = 4;
const byte COLS = 4;
char keys[ROWS][COLS] = {
  {'1','2','3','A'},
  {'2','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};
byte rowPins[ROWS] = {12, 14, 27, 26}; // Row pinouts
byte colPins[COLS] = {25, 33, 32, 35}; // Column pinouts

Keypad customKeypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);

// Passcode parameters
String correctCode = "1234";
String inputBuffer = "";
bool isLocked = true;

unsigned long unlockTimer = 0;
const unsigned long UNLOCK_DURATION_MS = 5000; // Unlock for 5 seconds

// Rolling Access Logs (Stores last 5 entries)
String accessLogs[5] = {"", "", "", "", ""};
int logCount = 0;

void addAccessLog(String message) {
  for (int i = 4; i > 0; i--) {
    accessLogs[i] = accessLogs[i - 1];
  }
  unsigned long timestampS = millis() / 1000;
  accessLogs[0] = "[" + String(timestampS) + "s] " + message;
  
  if (logCount < 5) logCount++;
  Serial.println("[SECURITY] " + accessLogs[0]);
}

// Unlock door function
void unlockDoor(String source) {
  isLocked = false;
  lockServo.write(90); // Rotate to open position
  unlockTimer = millis();
  addAccessLog("Access Granted (" + source + ")");
}

// Lock door function
void lockDoor() {
  isLocked = true;
  lockServo.write(0); // Rotate back to lock position
  addAccessLog("Door Re-locked");
}

// HTTP API endpoint returning JSON data
void handleGetLockData() {
  String lockState = isLocked ? "LOCKED" : "UNLOCKED";
  String json = "{\"state\":\"" + lockState + "\",\"logs\":[";
  for (int i = 0; i < 5; i++) {
    json += "\"" + accessLogs[i] + "\"";
    if (i < 4) json += ",";
  }
  json += "]}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to remote unlock
void handlePostUnlock() {
  unlockDoor("Web Remote Override");
  server.send(200, "text/plain", "OK");
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Smart Lock Dashboard</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .status-badge { display: inline-block; padding: 10px 20px; border-radius: 20px; font-weight: bold; font-size: 16px; text-transform: uppercase; margin: 20px 0; }\n";
  html += "  .locked { background-color: #991b1b; color: #fee2e2; }\n";
  html += "  .unlocked { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 14px; font-size: 16px; font-weight: bold; color: white; background-color: #38bdf8; border: none; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; }\n";
  html += "  .btn:hover { background-color: #0ea5e9; }\n";
  html += "  table { width: 100%; border-collapse: collapse; margin-top: 25px; text-align: left; font-size: 13px; }\n";
  html += "  th { color: #64748b; border-bottom: 2px solid #334155; padding-bottom: 8px; }\n";
  html += "  td { padding: 8px 0; border-bottom: 1px solid #1e293b; color: #94a3b8; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Keypad Security Lock</h1>\n";
  html += "  <div id=\"lockBadge\" class=\"status-badge locked\">LOCKED</div>\n";
  
  html += "  <button class=\"btn\" onclick=\"remoteUnlock()\">Remote Unlock</button>\n";
  
  html += "  <h3>Recent Access Logs</h3>\n";
  html += "  <table>\n";
  html += "    <thead><tr><th>Activity Logs</th></tr></thead>\n";
  html += "    <tbody id=\"logTable\"></tbody>\n";
  html += "  </table>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 500 ms | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateLockStatus() {\n";
  html += "    fetch('/api/lock')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        const bdgEl = document.getElementById('lockBadge');\n";
  html += "        bdgEl.innerText = data.state;\n";
  
  html += "        if (data.state === 'LOCKED') {\n";
  html += "          bdgEl.className = 'status-badge locked';\n";
  html += "        } else {\n";
  html += "          bdgEl.className = 'status-badge unlocked';\n";
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
  
  html += "  function remoteUnlock() {\n";
  html += "    fetch('/api/unlock', { method: 'POST' })\n";
  html += "      .then(() => updateLockStatus());\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateLockStatus();\n";
  html += "    setInterval(updateLockStatus, 500); // Poll twice a second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  lockServo.attach(SERVO_PIN);
  lockServo.write(0); // Initialize lock CLOSED
  
  Serial.println("\nESP32 Security Lock Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/lock", handleGetLockData);
  server.on("/api/unlock", HTTP_POST, handlePostUnlock);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  addAccessLog("Security System Armed");
}

void loop() {
  server.handleClient();
  
  // 1. Scan keypad for button clicks
  char key = customKeypad.getKey();
  
  if (key) {
    Serial.printf("[Keypad] Clicked key: %c\n", key);
    
    // Check if user clears input using '*' key
    if (key == '*') {
      inputBuffer = "";
      Serial.println("[Keypad] Input cleared.");
    } 
    // Check if user submits passcode using '#' key
    else if (key == '#') {
      if (inputBuffer == correctCode) {
        unlockDoor("Correct Passcode");
      } else {
        addAccessLog("Access Denied - Invalid Passcode (" + inputBuffer + ")");
        inputBuffer = ""; // Reset buffer
      }
    } 
    // Otherwise, append character to buffer (limit length to 8 characters)
    else {
      if (inputBuffer.length() < 8) {
        inputBuffer += key;
      }
    }
  }
  
  // 2. Non-blocking lock timeout manager
  if (!isLocked && (millis() - unlockTimer >= UNLOCK_DURATION_MS)) {
    lockDoor();
  }
  
  delay(1);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Keypad**, and **Servo** onto the canvas.
2. Wire Keypad rows/cols to **GPIO pins** matching the pinout tables, and Servo to **GPIO13**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Click keys `1`, `2`, `3`, `4`, and `#` on the keypad widget. Verify that the servo rotates to 90 degrees (unlocked), and the web dashboard logs the access.
6. Wait 5 seconds. Verify that the servo returns to 0 degrees.
7. Click the "Remote Unlock" button on the webpage. Verify that the servo opens.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[SECURITY] [0s] Security System Armed
[Keypad] Clicked key: 1
[Keypad] Clicked key: 2
[Keypad] Clicked key: 3
[Keypad] Clicked key: 4
[Keypad] Clicked key: #
[SECURITY] [12s] Access Granted (Correct Passcode)
[SECURITY] [17s] Door Re-locked
```

## Expected Canvas Behavior
* Pressing the correct keypad buttons rotates the servo widget to 90 degrees.
* The servo returns to 0 degrees after 5 seconds of inactivity.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `customKeypad.getKey()` | Scans the row and column matrix pins for closed contacts. |
| `key == '#'` | Standard confirmation key to submit passcode. |
| `inputBuffer == correctCode` | Compares the input buffer against the passcode. |
| `lockServo.write(90)` | Drives the lock bolt servo to the open position. |

## Hardware & Safety Concept: Passcode Length Restrictions and Brute Force Lockout
Matrix keypads use row-column grids to scan pins with minimal I/O pins (8 pins for 16 keys). To secure the system against brute-force attacks:
1. **Passcode Hashing**: Avoid storing passcodes in plain text in production.
2. **Brute Force Lockout**: Count consecutive incorrect passcode entries. If a user enters an incorrect passcode 3 times in a row, lock the keypad and disable inputs for 5 minutes, flashing an alert on the web dashboard.

## Try This! (Challenges)
1. **OLED Status Display**: Upgrade the screen to an SSD1306 OLED (Project 60) and display a lock icon showing if the door is open.
2. **Access denial buzzer**: Sound a long warning buzz on a buzzer (GPIO 15) when an incorrect passcode is entered.
3. **SPIFFS passcode configuration**: Save the passcode to SPIFFS and allow changing it from the web console.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Keypad clicks are ignored | Pin mapping wrong | Confirm that row and column pins match the pin configurations in the code |
| Servo does not move | Power rails missing | Confirm that the servo is powered from the 5V Vin rail, not the 3.3V rail |
| Keypad enters multiple digits on one press | Debounce missing | The Keypad library handles debouncing, verify that `delay(1)` is called in the loop to prevent glitches |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [56 - ESP32 Web Page Servo angle controller](56-esp32-web-page-servo-angle-controller-oled-display-feedback.md)
- [113 - ESP32 DS3231 RTC Alarm System IoT Control](113-esp32-ds3231-rtc-alarm-system-iot-control.md)
- [115 - ESP32 Keypad Web Display HUD](115-esp32-keypad-web-display-hud-keypad-lcd-web-chat-terminal.md) (Next project)
