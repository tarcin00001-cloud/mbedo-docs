# 199 - 4x4 Password Lock with Attempt Lockout, Local Siren, & IoT Email Alert

Build a high-security physical entry keypad lock on the ESP32 that checks passcodes on a 4x4 matrix keypad, controls a servo door latch on GPIO 15, locks out input after three failed attempts, triggers a buzzer siren, and alerts administrators via secure SMTP email.

## Goal
Learn how to interface 4x4 matrix keypads, implement state-machine lockouts, compute time debounces, compile SMTP email notifications, drive servo door locks, and build admin overrides.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A 4x4 Keypad is scanned by GPIO pins. If a user inputs the correct code (`1379#`), a servo motor on GPIO 15 rotates to unlock the door. If they enter three incorrect codes in a row, the ESP32 locks down the keypad for 5 minutes, sounds a warbling siren on a buzzer (GPIO 4), blinks a status LED on GPIO 2, and sends an email alert. The administrator can monitor entry logs and override the lockout from a local web browser.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| Active/Passive Buzzer | `buzzer` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Keypad | Row 1 - Row 4 | GPIO13 / GPIO12 / GPIO14 / GPIO27 | Yellow / Orange / Blue / Green | Keypad row scan lines |
| Keypad | Col 1 - Col 4 | GPIO26 / GPIO25 / GPIO33 / GPIO32 | Purple / Brown / Grey / White | Keypad column scan lines |
| Servo Motor | PWM (Signal) | GPIO15 | Orange | Door lock servo control |
| Buzzer | Positive (+) | GPIO4 | Blue | Alarm sounder |
| LED | Anode (+) | GPIO2 | Red | Lockout warning indicator |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 220 Ω resistor in series with the LED, and a 100 Ω resistor with the buzzer.

## Code
```cpp
// 4x4 Password Lock (Keypad scan + Attempt counter + Latch Lock + SMTP SSL + Web overrides)
#include <WiFi.h>
#include <WebServer.h>
#include <Keypad.h>
#include <ESP32Servo.h>
#include <ESP_Mail_Client.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// SMTP Server Configuration (Example using Gmail SMTP)
#define SMTP_HOST "smtp.gmail.com"
#define SMTP_PORT 465
#define SENDER_EMAIL "myaddress@gmail.com"
#define SENDER_PASSWORD "xxxx xxxx xxxx xxxx" // App Password
#define RECIPIENT_EMAIL "security@example.com"

// Keypad Configuration
const byte ROWS = 4;
const byte COLS = 4;
char keys[ROWS][COLS] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};
byte rowPins[ROWS] = {13, 12, 14, 27};
byte colPins[COLS] = {26, 25, 33, 32};
Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);

// Master Passcode (Must end with # to submit)
const String masterCode = "1379";
String enteredCode = "";

Servo lockServo;

const int SERVO_PIN = 15;
const int BUZZER_PIN = 4;
const int LED_PIN = 2;

WebServer server(80);
SMTPSession smtp;

// Lock States
bool isLocked = true;
int failedAttempts = 0;
bool lockoutActive = false;
unsigned long lockoutStartTime = 0;
const unsigned long LOCKOUT_DURATION_MS = 300000; // 5-minute keypad lockout

// Callback monitoring SMTP transmission status
void smtpCallback(SMTP_Status status) {
  Serial.println(status.info());
}

// Send intruder alert email using SMTP SSL
void sendSecurityAlertEmail() {
  if (WiFi.status() == WL_CONNECTED) {
    ESP_Mail_Session session;
    session.server.host_name = SMTP_HOST;
    session.server.port = SMTP_PORT;
    session.login.user_name = SENDER_EMAIL;
    session.login.password = SENDER_PASSWORD;
    
    SMTP_Message message;
    message.sender.name = "Front Door Security Node";
    message.sender.email = SENDER_EMAIL;
    message.subject = "🚨 EXTREME ALERT: Brute Force Keypad Lockout!";
    message.addRecipient("Security Team", RECIPIENT_EMAIL);
    
    // HTML Message Body
    String htmlMsg = "<div style='font-family:sans-serif;border:2px solid #ef4444;padding:25px;border-radius:10px;'>";
    htmlMsg += "<h2 style='color:#ef4444;margin-top:0;'>SECURITY BREACH WARNING</h2>";
    htmlMsg += "<p>A brute force attack has been detected on the Front Door entry keypad.</p>";
    htmlMsg += "<ul>";
    htmlMsg += "  <li><strong>Failed Attempts:</strong> 3 consecutive failures</li>";
    htmlMsg += "  <li><strong>Action Taken:</strong> 5-minute keypad lockout initiated</li>";
    htmlMsg += "  <li><strong>Lockout State:</strong> ACTIVE</li>";
    htmlMsg += "  <li><strong>Device Uptime:</strong> " + String(millis() / 1000) + " seconds</li>";
    htmlMsg += "</ul>";
    htmlMsg += "<p style='color:#b91c1c;font-weight:bold;'>System lockdown active. Inspect local security cameras.</p>";
    htmlMsg += "</div>";
    
    message.html.content = htmlMsg.c_str();
    message.html.charSet = "utf-8";
    
    if (!smtp.connect(&session)) {
      Serial.println("[SMTP Error] Connection failed!");
      return;
    }
    
    if (!MailClient.sendMail(&smtp, &message)) {
      Serial.println("[SMTP Error] Sending failed: " + smtp.errorReason());
    }
  }
}

// Unlock door latch
void unlockDoor() {
  isLocked = false;
  lockServo.write(90); // 90 degrees represents unlocked position
  digitalWrite(LED_PIN, LOW);
  failedAttempts = 0;
  
  // Open chime
  tone(BUZZER_PIN, 1200, 100);
  delay(150);
  tone(BUZZER_PIN, 1500, 100);
  
  Serial.println("[Security] Passcode correct. Door UNLOCKED.");
}

// Lock door latch
void lockDoor() {
  isLocked = true;
  lockServo.write(0); // 0 degrees represents locked position
  Serial.println("[Security] Door LOCKED.");
}

// Trigger lockout alarm
void triggerLockout() {
  lockoutActive = true;
  lockoutStartTime = millis();
  lockDoor();
  
  Serial.println("[Security Alert] Three failed entry attempts. Lockout active.");
  
  // Send email warning
  sendSecurityAlertEmail();
}

// Reset lockout state
void resetLockout() {
  lockoutActive = false;
  failedAttempts = 0;
  digitalWrite(LED_PIN, LOW);
  noTone(BUZZER_PIN);
  Serial.println("[Security] Lockout cleared. Keypad active.");
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Security Portal HUD</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 480px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .status-box { text-align: center; padding: 18px; border-radius: 8px; margin: 20px 0; font-weight: bold; font-size: 16px; border: 1px solid #334155; }\n";
  html += "  .locked { background-color: #7f1d1d; color: #fca5a5; }\n";
  html += "  .unlocked { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .lockout { background-color: #b45309; color: #fef3c7; animation: blink 1s infinite; }\n";
  html += "  .info-text { font-size: 13px; color: #94a3b8; margin: 20px 0; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 12px; margin-bottom: 10px; font-weight: bold; border-radius: 6px; border: none; cursor: pointer; text-align: center; text-decoration: none; font-size: 14px; }\n";
  html += "  .btn-unlock { background-color: #10b981; color: white; }\n";
  html += "  .btn-lock { background-color: #3b82f6; color: white; }\n";
  html += "  .btn-reset { background-color: #ef4444; color: white; }\n";
  html += "  @keyframes blink { 50% { opacity: 0.8; } }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Entry Console HUD</h1>\n";
  
  // Status check
  if (lockoutActive) {
    html += "  <div class=\"status-box lockout\">⚠️ KEYPAD LOCKED OUT</div>\n";
  } else if (isLocked) {
    html += "  <div class=\"status-box locked\">DOOR LOCKED</div>\n";
  } else {
    html += "  <div class=\"status-box unlocked\">DOOR UNLOCKED</div>\n";
  }
  
  html += "  <div class=\"info-text\">\n";
  html += "    Failed Attempts: " + String(failedAttempts) + "/3<br>\n";
  if (lockoutActive) {
    unsigned long remainingSec = (LOCKOUT_DURATION_MS - (millis() - lockoutStartTime)) / 1000;
    html += "    Lockout Remaining: " + String(remainingSec) + " seconds\n";
  }
  html += "  </div>\n";
  
  // Actions
  if (lockoutActive) {
    html += "  <a href=\"/reset\" class=\"btn btn-reset\">RESET SYSTEM LOCKOUT</a>\n";
  } else {
    html += "  <a href=\"/toggle\" class=\"btn " + String(isLocked ? "btn-unlock" : "btn-lock") + "\">";
    html += isLocked ? "REMOTE UNLOCK DOOR" : "REMOTE LOCK DOOR";
    html += "  </a>\n";
  }
  
  html += "</div>\n";
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// Remote lock/unlock route
void handleToggle() {
  if (!lockoutActive) {
    if (isLocked) unlockDoor(); else lockDoor();
  }
  server.sendHeader("Location", "/", true);
  server.send(303);
}

// Remote reset route
void handleReset() {
  resetLockout();
  server.sendHeader("Location", "/", true);
  server.send(303);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  noTone(BUZZER_PIN);
  
  // Attach Servo
  lockServo.attach(SERVO_PIN);
  lockServo.write(0); // Lock door on startup
  
  Serial.println("\nESP32 Entry Lockout Station starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  Serial.print("Local IP Address: ");
  Serial.println(WiFi.localIP());
  
  // Server Routes
  server.on("/", HTTP_GET, handleRoot);
  server.on("/toggle", HTTP_GET, handleToggle);
  server.on("/reset", HTTP_GET, handleReset);
  server.begin();
  
  smtp.callback(smtpCallback);
}

void loop() {
  server.handleClient();
  
  // 1. Evaluate Lockout Cooldown timer
  if (lockoutActive) {
    // Sound local pulsing alarm
    static unsigned long lastChime = 0;
    if (millis() - lastChime > 500) {
      lastChime = millis();
      digitalWrite(LED_PIN, !digitalRead(LED_PIN));
      tone(BUZZER_PIN, digitalRead(LED_PIN) ? 1500 : 800, 200);
    }
    
    // Auto-reset lockout after duration
    if (millis() - lockoutStartTime >= LOCKOUT_DURATION_MS) {
      resetLockout();
    }
  } 
  // 2. Scan Keypad if lockout is not active
  else {
    char key = keypad.getKey();
    
    if (key) {
      Serial.printf("[Keypad Input] Pressed: %c\n", key);
      
      if (key == '#') {
        // Evaluate passcode on '#' keypress
        if (enteredCode == masterCode) {
          unlockDoor();
        } else {
          failedAttempts++;
          playChime(false);
          Serial.printf("[Security Error] Incorrect code. Failed attempts: %d/3\n", failedAttempts);
          
          if (failedAttempts >= 3) {
            triggerLockout();
          }
        }
        enteredCode = ""; // Reset buffer
      } 
      else if (key == '*') {
        enteredCode = ""; // Clear buffer
        tone(BUZZER_PIN, 1000, 100);
      } 
      else {
        enteredCode += key;
        tone(BUZZER_PIN, 1000, 50); // Play brief key click beep
      }
    }
    
    // Auto-lock door after 6 seconds
    static unsigned long lastUnlockTime = 0;
    if (!isLocked) {
      if (lastUnlockTime == 0) lastUnlockTime = millis();
      if (millis() - lastUnlockTime > 6000) {
        lockDoor();
        lastUnlockTime = 0;
      }
    }
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **4x4 Keypad**, **Servo**, **Buzzer**, and **LED** onto the canvas.
2. Wire Row pins to **13, 12, 14, 27** and Col pins to **26, 25, 33, 32**. Wire Servo to **15**, Buzzer to **4**, and LED anode to **2**.
3. Paste the code and click **Run**.
4. In simulation, type the passcode keys `1`, `3`, `7`, `9`, followed by `#` on the virtual keypad widget.
5. Verify that the servo motor rotates to 90 degrees and the buzzer plays two ascending tones.
6. Let the lock reset. Now, enter incorrect key sequences (e.g. `2`, `4`, `6`, `#`) three times.
7. Verify that the buzzer starts a loud pulsing alarm, the LED flashes, and the console shows the lockout trigger message.
8. Open your web browser, navigate to the IP address, and click "RESET SYSTEM LOCKOUT" to restore functionality.

## Expected Output
Serial Monitor:
```
WiFi Connected.
Local IP Address: 10.10.0.3
[Keypad Input] Pressed: 1
[Keypad Input] Pressed: 3
[Keypad Input] Pressed: 7
[Keypad Input] Pressed: 9
[Keypad Input] Pressed: #
[Security] Passcode correct. Door UNLOCKED.
[Security Error] Incorrect code. Failed attempts: 3/3
[Security Alert] Three failed entry attempts. Lockout active.
[SMTP] Connecting to server...
[SMTP] Email sent successfully!
```

## Expected Canvas Behavior
* Pressing keypads sends inputs. Entering three incorrect codes drives the buzzer widget, flashes the LED, locks out input, and generates emails.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `keypad.getKey()` | Scans the row and column matrix pins, resolving active button closures. |
| `failedAttempts >= 3` | Evaluates if the system has reached the brute-force threshold to lock. |
| `LOCKOUT_DURATION_MS` | Defines the duration the keypad remains disabled during lockdowns. |

## Hardware & Safety Concept: Keypad Matrix Ghosting and Lockbox Solenoids
* **Keypad Matrix Ghosting**: When multiple keys are pressed simultaneously on a matrix keypad, the scanning pins can register false keypresses (ghosting). In entry locks, ignore multiple keys by ignoring inputs if more than one column goes LOW simultaneously during scans.
* **Lockbox Solenoids**: Servos provide rotation, but physical security bolts usually use heavy DC Solenoids. Solenoids draw huge current spikes (up to 2–3 Amps for 12V frames). Always drive solenoids using a flyback diode-protected MOSFET switch (e.g., IRF540N) powered by an external DC supply, isolating the ESP32.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display "Enter Passcode:" and a countdown during lockout.
2. **Access Card RFID Integration**: Combine this project with Project 193 to require both a correct keypad passcode and a valid RFID card scan (multi-factor authentication).
3. **Change passcode online**: Create a secure web interface page that lets administrators update the master passcode dynamically, saving it in NVS.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Keypad registers wrong keys | Row/Column wiring swapped | Double-check that your keypad row and column pins match the C++ constructor array definitions exactly |
| Servo stutters or drops out | High current draw | Servos require stable power. Do not power the servo from the ESP32's 3V3 output; always use the 5V/VIN pin or an external 5V regulator |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [114 - Keypad 4x4 Password Lock IoT](../intermediate/114-keypad-4x4-password-lock-iot.md)
- [193 - RFID Vault with Acceleration Safeguards & IoT Event logging](193-rfid-vault-with-acceleration-safeguards-iot-event-logging.md)
- [200 - Smart GPS Autonomous Waypoint Tracker with Web Map visualizer](200-smart-gps-autonomous-waypoint-tracker-with-web-map-visualizer.md) (Next project)
