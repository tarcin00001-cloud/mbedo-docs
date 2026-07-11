# 158 - Keypad RFID Multi-factor Lock IoT with Remote Open URL override

Build a multi-factor authentication (MFA) smart lock system on the ESP32 that requires both a valid MFRC522 RFID card scan and a correct 4-digit PIN code entered via a 4x4 matrix keypad, controls a lock servo on GPIO 2, and hosts a web dashboard with status logs and administrative unlock overrides.

## Goal
Learn how to implement multi-sensor state machines, handle sequential multi-factor authentication logic, parse matrix keypad arrays, control servo motors, and construct administrative web overrides.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An MFRC522 RFID reader is on SPI, a 4x4 Keypad on GPIOs, and a lock servo on GPIO 2. The user must first swipe an authorized RFID card (`4A8F9C2D`). If valid, the system prompts for a PIN. Entering `1234` within 10 seconds unlocks the door (servo rotates to 90°). Navigating to the ESP32's IP address displays a dashboard with status logs and a button to trigger an administrative override unlock.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MFRC522 RFID Reader | `rfid` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| RFID Module | SDA (SS) | GPIO5 | Yellow | SPI chip select |
| RFID Module | SCK | GPIO18 | Green | SPI clock |
| RFID Module | MISO | GPIO19 | Blue | SPI master input |
| RFID Module | MOSI | GPIO23 | White | SPI master output |
| RFID Module | RST | GPIO14 | Orange | Reset input |
| Keypad Rows | R1 - R4 | GPIO12 / 13 / 25 / 26 | Blue / White | Row outputs |
| Keypad Cols | C1 - C4 | GPIO27 / 32 / 33 / 15 | Purple / Grey | Column inputs |
| Servo Motor | PWM (Signal) | GPIO2 | Red | Lock actuator signal |
| Buzzer | Positive (+) | GPIO4 | Green | Audio feedback signal |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Keypads require internal pull-ups on the column pins. Power the RFID reader from the 3.3V rail and the servo from the 5V rail.

## Code
```cpp
// Keypad RFID Multi-factor Lock IoT (MFA State Machine + SPI RFID + 4x4 Keypad + Web Unlock)
#include <WiFi.h>
#include <WebServer.h>
#include <SPI.h>
#include <MFRC522.h>
#include <Keypad.h>
#include <ESP32Servo.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// RFID Pins
const int RFID_SS_PIN = 5;
const int RFID_RST_PIN = 14;
MFRC522 mfrc522(RFID_SS_PIN, RFID_RST_PIN);

// Keypad Configuration
const byte ROWS = 4;
const byte COLS = 4;
char keys[ROWS][COLS] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};
byte rowPins[ROWS] = {12, 13, 25, 26};
byte colPins[COLS] = {27, 32, 33, 15};
Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);

const int SERVO_PIN = 2;
const int BUZZER_PIN = 4;
Servo lockServo;

WebServer server(80);

// Authorized Credentials
const String AUTH_RFID = "4A8F9C2D";
const String AUTH_PIN = "1234";

// Multi-Factor State Machine
enum LockState {
  STATE_LOCKED,
  STATE_WAITING_PIN,
  STATE_UNLOCKED
};
LockState currentState = STATE_LOCKED;

String pinBuffer = "";
unsigned long stateTransitionTime = 0;
const unsigned long PIN_TIMEOUT_MS = 10000; // 10-second timeout for PIN entry

// Log buffer (Stores last 5 entries)
String accessLogs[5] = {"", "", "", "", ""};
int logCount = 0;

void addLog(String message) {
  for (int i = 4; i > 0; i--) {
    accessLogs[i] = accessLogs[i - 1];
  }
  unsigned long timestampS = millis() / 1000;
  accessLogs[0] = "[" + String(timestampS) + "s] " + message;
  if (logCount < 5) logCount++;
  Serial.println("[Security] " + accessLogs[0]);
}

void soundBeep(int freq, int duration) {
  tone(BUZZER_PIN, freq);
  delay(duration);
  noTone(BUZZER_PIN);
}

// Access Unlock trigger
void unlockLock(String method) {
  currentState = STATE_UNLOCKED;
  addLog("Unlocked via " + method);
  soundBeep(1200, 100);
  soundBeep(1500, 150);
  
  lockServo.write(90); // Rotate servo to 90 degrees
  delay(3000);        // Hold open for 3 seconds
  
  lockServo.write(0);  // Relock
  currentState = STATE_LOCKED;
  pinBuffer = "";
  addLog("Auto-Relocked");
  soundBeep(800, 200);
}

// HTTP API endpoint returning JSON data
void handleGetStatus() {
  String stateStr = "LOCKED";
  if (currentState == STATE_WAITING_PIN) stateStr = "WAITING PIN";
  else if (currentState == STATE_UNLOCKED) stateStr = "UNLOCKED";
  
  String json = "{\"state\":\"" + stateStr + "\"" +
                 ",\"logs\":[";
  for (int i = 0; i < 5; i++) {
    json += "\"" + accessLogs[i] + "\"";
    if (i < 4) json += ",";
  }
  json += "]}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to trigger administrative override unlock
void handlePostUnlock() {
  addLog("Admin Override Request Received");
  server.send(200, "text/plain", "OK");
  
  // Trigger unlock in a separate thread/step
  unlockLock("Admin Web Panel");
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Secure Access Controller</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .status-badge { display: inline-block; padding: 10px 20px; border-radius: 20px; font-weight: bold; font-size: 15px; text-transform: uppercase; margin: 20px 0; background-color: #7f1d1d; color: #fca5a5; }\n";
  html += "  .status-badge.unlocked { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .status-badge.waiting { background-color: #92400e; color: #fef3c7; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 14px; font-size: 16px; font-weight: bold; color: white; background-color: #38bdf8; border: none; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; margin-top: 10px; }\n";
  html += "  .btn:hover { background-color: #0ea5e9; }\n";
  html += "  table { width: 100%; border-collapse: collapse; margin-top: 25px; text-align: left; font-size: 13px; }\n";
  html += "  th { color: #64748b; border-bottom: 2px solid #334155; padding-bottom: 8px; }\n";
  html += "  td { padding: 8px 0; border-bottom: 1px solid #1e293b; color: #94a3b8; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>MFA Access Panel</h1>\n";
  html += "  <div id=\"statusDisplay\" class=\"status-badge\">LOCKED</div>\n";
  
  html += "  <button class=\"btn\" onclick=\"adminUnlock()\">Force Unlock Override</button>\n";
  
  html += "  <h3>MFA Access Logs</h3>\n";
  html += "  <table>\n";
  html += "    <thead><tr><th>Log Entry</th></tr></thead>\n";
  html += "    <tbody id=\"logTable\"></tbody>\n";
  html += "  </table>\n";
  
  html += "  <p class=\"footer\">Multi-factor Lock active | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateHUD() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        const bdg = document.getElementById('statusDisplay');\n";
  html += "        bdg.innerText = data.state;\n";
  
  html += "        if(data.state === 'UNLOCKED') {\n";
  html += "          bdg.className = 'status-badge unlocked';\n";
  html += "        } else if(data.state === 'WAITING PIN') {\n";
  html += "          bdg.className = 'status-badge waiting';\n";
  html += "        } else {\n";
  html += "          bdg.className = 'status-badge';\n";
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
  
  html += "  function adminUnlock() {\n";
  html += "    fetch('/api/unlock', { method: 'POST' })\n";
  html += "      .then(() => updateHUD());\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateHUD();\n";
  html += "    setInterval(updateHUD, 1000);\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  SPI.begin();
  mfrc522.PCD_Init();
  
  lockServo.attach(SERVO_PIN);
  lockServo.write(0); // LOCKED
  
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("\nMFA Access Controller Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/status", handleGetStatus);
  server.on("/api/unlock", HTTP_POST, handlePostUnlock);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  addLog("MFA System Online");
}

void loop() {
  server.handleClient();
  
  // 1. Process MFA State Machine Transitions
  unsigned long now = millis();
  
  // Reset state to LOCKED if PIN entry times out
  if (currentState == STATE_WAITING_PIN && (now - stateTransitionTime >= PIN_TIMEOUT_MS)) {
    currentState = STATE_LOCKED;
    pinBuffer = "";
    addLog("PIN entry timeout. Lock Reset.");
    soundBeep(400, 400);
  }
  
  // 2. Read RFID Card Swipe (only valid in LOCKED state)
  if (currentState == STATE_LOCKED && mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
    String cardUid = "";
    for (byte i = 0; i < mfrc522.uid.size; i++) {
      cardUid += String(mfrc522.uid.uidByte[i] < 0x10 ? "0" : "");
      cardUid += String(mfrc522.uid.uidByte[i], HEX);
    }
    cardUid.toUpperCase();
    
    Serial.printf("\n[RFID] Scanned card: %s\n", cardUid.c_str());
    
    if (cardUid == AUTH_RFID) {
      currentState = STATE_WAITING_PIN;
      stateTransitionTime = now;
      addLog("Card OK. Awaiting PIN code.");
      soundBeep(1000, 100);
    } else {
      addLog("Invalid Card Swiped: " + cardUid);
      soundBeep(400, 400);
    }
    
    mfrc522.PICC_HaltA();
  }
  
  // 3. Read Keypad Entries (only valid in WAITING PIN state)
  if (currentState == STATE_WAITING_PIN) {
    char key = keypad.getKey();
    if (key != NO_KEY) {
      soundBeep(1000, 50);
      
      // Cancel PIN entry if '*' is pressed
      if (key == '*') {
        currentState = STATE_LOCKED;
        pinBuffer = "";
        addLog("PIN cancelled by user.");
        soundBeep(400, 300);
      } 
      // Submit PIN if '#' is pressed or once 4 digits are accumulated
      else {
        pinBuffer += key;
        
        if (pinBuffer.length() >= 4) {
          if (pinBuffer == AUTH_PIN) {
            unlockLock("RFID + PIN");
          } else {
            currentState = STATE_LOCKED;
            pinBuffer = "";
            addLog("Incorrect PIN entered. Lock Reset.");
            soundBeep(400, 500);
          }
        }
      }
    }
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MFRC522 RFID Module**, **4x4 Keypad**, **Servo Motor**, and **Buzzer** onto the canvas.
2. Wire RFID to SPI pins (CS to **GPIO5**, SCK to **GPIO18**, MISO to **GPIO19**, MOSI to **GPIO23**, RST to **GPIO14**).
3. Wire Keypad rows to **GPIO12, 13, 25, 26** and columns to **GPIO27, 32, 33, 15**.
4. Wire Servo to **GPIO2** and Buzzer to **GPIO4**.
5. Paste the code and click **Run**.
6. Open your web browser and navigate to the printed IP address.
7. Click the RFID widget, select card `4A8F9C2D`, and swipe it. Verify that the webpage status updates to `WAITING PIN`.
8. Click digits `1`, `2`, `3`, and `4` sequentially on the simulated keypad. Verify that the servo rotates to 90 degrees and relocks after 3 seconds.
9. Click "Force Unlock Override" on the webpage. Verify that the servo rotates to 90 degrees instantly.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Security] [0s] MFA System Online
[RFID] Scanned card: 4A8F9C2D
[Security] [12s] Card OK. Awaiting PIN code.
[Security] [18s] Unlocked via RFID + PIN
[Security] [21s] Auto-Relocked
```

## Expected Canvas Behavior
* Swiping the correct card and entering the matching PIN rotates the servo widget and triggers the success chime.
* Webpage override buttons trigger administrative unlock immediately, bypassing local inputs.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `currentState = STATE_WAITING_PIN` | Transitions state machine to await PIN entry. |
| `now - stateTransitionTime >= PIN_TIMEOUT_MS` | Evaluates if the 10-second entry window has elapsed. |
| `pinBuffer == AUTH_PIN` | Compares PIN input against authorized credentials. |
| `unlockLock(...)` | Performs the unlocking sequence and returns to LOCKED. |

## Hardware & Safety Concept: MFA State Machine Security and Panic Locks
* **State Machine Security**: Traditional electronic locks are vulnerable: shorting the keypad signals can trick the controller into unlocking. By executing all logic inside a **closed state machine** (where key entries are ignored until a card swipe transitions the state to `STATE_WAITING_PIN`), physical tampering becomes much more difficult.
* **Panic Lockout**: If an incorrect PIN is entered 3 times in a row, the controller should enter a lockout state (e.g. ignoring all cards and key entries for 5 minutes) and publish a security alert (Project 148) to the administration dashboard.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a 4-digit password masking field (e.g., `****`) during entry.
2. **Master Key**: Program a specific master RFID card that bypasses the PIN step to unlock the door instantly.
3. **SPIFFS Database**: Load authorized UIDs and PIN codes from a JSON file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Keypad inputs are ignored | Pin collision or floating inputs | Ensure row pins are set as outputs and col pins are configured with internal pull-ups in the keypad driver |
| Lock unlocks on card swipe without PIN | State machine bug | Confirm that the state machine is implemented properly and requires both steps |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [114 - ESP32 Keypad 4x4 password lock with access logs](114-esp32-keypad-4x4-password-lock-iot-keypad-servo-web-access-logs.md)
- [143 - Firebase Authentication Access Controller](143-firebase-authentication-access-controller.md)
- [159 - Automatic Battery Capacity Tester IoT Web Logger](159-esp32-automatic-battery-capacity-tester-iot-web-logger.md) (Next project)
