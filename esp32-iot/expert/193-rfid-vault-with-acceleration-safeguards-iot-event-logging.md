# 193 - RFID Vault with Acceleration Safeguards & IoT Event logging

Build a physical security lock system on the ESP32 that uses an MFRC522 RFID reader for access control, an MPU-6050 accelerometer to detect physical tampering/theft attempts, a servo motor lock on GPIO 13, and a local SPIFFS audit logger accessible via a web dashboard.

## Goal
Learn how to interface SPI RFID readers, read MPU-6050 I2C accelerometer sensors, write event logs to SPIFFS, trigger servo locks, configure buzzer alarms, and serve interactive diagnostic web pages.

## What You Will Build
An ESP32 DevKitC acts as a vault lock controller. A user scans an RFID card. If the UID matches, a servo motor on GPIO 13 rotates to unlock the vault door, and the event is logged. Meanwhile, an MPU-6050 sensor monitors for physical tilt or drop attacks. If a vibration exceeding a safety threshold is detected, the vault goes into lockdown (denying all access), triggers a siren on a buzzer (GPIO 15), and updates the local SPIFFS log. The admin can view these logs and reset the alarm from a web browser.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MFRC522 RFID Reader | `rfid` | Yes | Yes |
| MPU-6050 Accelerometer/Gyro | `mpu6050` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| Active/Passive Buzzer | `buzzer` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 | SDA / SCK / MOSI / MISO | GPIO5 / GPIO18 / GPIO23 / GPIO19 | Yellow / Orange / Blue / Green | SPI communication pins |
| MFRC522 | RST / GND / 3.3V | GPIO22 / GND / 3V3 | Gray / Black / Red | Reset, Ground, Power |
| MPU-6050 | SDA / SCL | GPIO21 / GPIO22 | Purple / Brown | I2C communication pins |
| Servo Motor | PWM (Signal) | GPIO13 | Orange | Lock servo control |
| Buzzer | Positive (+) | GPIO15 | Blue | Alarm sounder |
| LED | Anode (+) | GPIO12 | Red | Status indicator |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 220 Ω resistor in series with the LED, and a 100 Ω resistor with the buzzer.

## Code
```cpp
// RFID Vault (SPI MFRC522 + I2C MPU-6050 + Servo Lock + SPIFFS Logger + Web Dashboard)
#include <WiFi.h>
#include <WebServer.h>
#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <ESP32Servo.h>
#include <SPIFFS.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// RFID Pins
#define SS_PIN 5
#define RST_PIN 22
MFRC522 rfid(SS_PIN, RST_PIN);

// Valid RFID UID (Replace with your actual card UID)
const String validUID = "DE AD BE EF";

Adafruit_MPU6050 mpu;
Servo lockServo;

const int SERVO_PIN = 13;
const int BUZZER_PIN = 15;
const int LED_PIN = 12;

WebServer server(80);

// Vault States
bool isLocked = true;
bool alarmActive = false;

// Vibration acceleration threshold (in m/s^2)
const float TAMPER_THRESHOLD = 15.0;

// Log an event to SPIFFS
void logEvent(String eventType, String details) {
  File logFile = SPIFFS.open("/audit.log", FILE_APPEND);
  if (logFile) {
    String logEntry = "[" + String(millis() / 1000) + "] " + eventType + " - " + details + "\n";
    logFile.print(logEntry);
    logFile.close();
    Serial.print("[Audit Log] " + logEntry);
  }
}

// Chime chimes
void playChime(bool success) {
  if (success) {
    tone(BUZZER_PIN, 1500, 150);
    delay(200);
    tone(BUZZER_PIN, 2000, 150);
  } else {
    tone(BUZZER_PIN, 800, 300);
    delay(400);
    tone(BUZZER_PIN, 600, 300);
  }
}

// Unlock vault door
void unlockVault() {
  isLocked = false;
  lockServo.write(90); // 90 degrees represents unlocked state
  digitalWrite(LED_PIN, HIGH);
  logEvent("ACCESS", "Vault unlocked via RFID card.");
  playChime(true);
}

// Lock vault door
void lockVault() {
  isLocked = true;
  lockServo.write(0); // 0 degrees represents locked state
  digitalWrite(LED_PIN, LOW);
  logEvent("LOCK", "Vault locked automatically.");
  noTone(BUZZER_PIN);
}

// Trigger tamper alarm
void triggerTamperAlarm(float accelValue) {
  alarmActive = true;
  isLocked = true;
  lockServo.write(0); // Force lock
  logEvent("TAMPER", "Acceleration spike detected: " + String(accelValue, 2) + " m/s^2. LOCKDOWN initiated.");
}

// Serve root webpage displaying status and logs
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Vault Control Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 500px; width: 100%; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; text-align: center; margin-top: 0; }\n";
  html += "  .status-box { text-align: center; padding: 20px; border-radius: 8px; margin: 20px 0; font-weight: bold; font-size: 18px; border: 1px solid #334155; }\n";
  html += "  .locked { background-color: #7f1d1d; color: #fca5a5; }\n";
  html += "  .unlocked { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .alarm { background-color: #b45309; color: #fef3c7; animation: blink 1s infinite; }\n";
  html += "  .log-box { height: 180px; overflow-y: auto; background-color: #0f172a; border-radius: 8px; border: 1px solid #334155; padding: 15px; font-family: monospace; font-size: 12px; color: #94a3b8; margin: 20px 0; white-space: pre-wrap; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 12px; margin-bottom: 10px; font-weight: bold; border-radius: 6px; border: none; cursor: pointer; text-align: center; text-decoration: none; }\n";
  html += "  .btn-unlock { background-color: #10b981; color: white; }\n";
  html += "  .btn-reset { background-color: #ef4444; color: white; }\n";
  html += "  .btn-clear { background-color: #334155; color: #94a3b8; }\n";
  html += "  @keyframes blink { 50% { opacity: 0.7; } }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Vault Security Console</h1>\n";
  
  // Status logic
  if (alarmActive) {
    html += "  <div class=\"status-box alarm\">⚠️ SECURITY ALARM ACTIVE</div>\n";
  } else if (isLocked) {
    html += "  <div class=\"status-box locked\">SECURED - LOCKED</div>\n";
  } else {
    html += "  <div class=\"status-box unlocked\">ACCESS GRANTED - UNLOCKED</div>\n";
  }
  
  // Display logs
  html += "  <div class=\"log-box\">";
  File logFile = SPIFFS.open("/audit.log", FILE_READ);
  if (logFile) {
    while (logFile.available()) {
      html += (char)logFile.read();
    }
    logFile.close();
  } else {
    html += "No logs found.";
  }
  html += "</div>\n";
  
  // Admin commands
  if (alarmActive) {
    html += "  <a href=\"/reset\" class=\"btn btn-reset\">RESET SECURITY LOCKOUT</a>\n";
  } else {
    html += "  <a href=\"/unlock\" class=\"btn btn-unlock\">REMOTE UNLOCK DOOR</a>\n";
  }
  html += "  <a href=\"/clear\" class=\"btn btn-clear\">CLEAR EVENT AUDIT LOGS</a>\n";
  
  html += "</div>\n";
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// Remote unlock GET handler
void handleRemoteUnlock() {
  if (!alarmActive) {
    unlockVault();
    server.sendHeader("Location", "/", true);
    server.send(303);
  } else {
    server.send(403, "text/plain", "Action Denied. Vault in Lockdown.");
  }
}

// Reset alarm GET handler
void handleReset() {
  alarmActive = false;
  lockVault();
  logEvent("SYSTEM", "Alarm reset by administrator.");
  server.sendHeader("Location", "/", true);
  server.send(303);
}

// Clear log GET handler
void handleClear() {
  SPIFFS.remove("/audit.log");
  logEvent("SYSTEM", "Audit logs cleared.");
  server.sendHeader("Location", "/", true);
  server.send(303);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  
  // Initialize SPI & MFRC522 RFID reader
  SPI.begin();
  rfid.PCD_Init();
  
  // Initialize I2C and MPU-6050 Accelerometer
  Wire.begin();
  if (!mpu.begin()) {
    Serial.println("[MPU-6050 Error] Device not found!");
  } else {
    Serial.println("[MPU-6050] Connected successfully.");
  }
  
  // Initialize Servo
  lockServo.attach(SERVO_PIN);
  lockServo.write(0); // Initial state locked (0 degrees)
  
  // Initialize SPIFFS File system
  if (!SPIFFS.begin(true)) {
    Serial.println("[SPIFFS Error] Mount failed!");
  } else {
    Serial.println("[SPIFFS] Mounted successfully.");
  }
  
  Serial.println("\nESP32 Security Vault starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  Serial.print("Local IP Address: ");
  Serial.println(WiFi.localIP());
  
  // Web Server Routes
  server.on("/", handleRoot);
  server.on("/unlock", handleRemoteUnlock);
  server.on("/reset", handleReset);
  server.on("/clear", handleClear);
  server.begin();
  
  logEvent("SYSTEM", "System armed and active.");
}

void loop() {
  server.handleClient();
  
  // 1. TAMPER DETECTOR: Read acceleration values from MPU-6050
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);
  
  // Calculate total acceleration magnitude (resultant vector)
  float totalAccel = sqrt(a.acceleration.x * a.acceleration.x + 
                          a.acceleration.y * a.acceleration.y + 
                          a.acceleration.z * a.acceleration.z);
                          
  // If acceleration magnitude spikes (excluding normal gravity ~9.8 m/s^2)
  if (abs(totalAccel - 9.8) > TAMPER_THRESHOLD && !alarmActive) {
    triggerTamperAlarm(totalAccel);
  }
  
  // 2. Local alarm sound loop
  if (alarmActive) {
    // Sound warble siren
    tone(BUZZER_PIN, 1800);
    delay(100);
    tone(BUZZER_PIN, 1200);
    delay(100);
  }
  
  // 3. RFID ACCESS CONTROLLER (Only check if alarm is not active)
  if (!alarmActive) {
    // Auto-lock door after 5 seconds
    static unsigned long unlockStartTime = 0;
    if (!isLocked && millis() - unlockStartTime > 5000) {
      lockVault();
    }
    
    // Look for new RFID cards
    if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
      // Decode card UID to String format
      String readUID = "";
      for (byte i = 0; i < rfid.uid.size; i++) {
        readUID += String(rfid.uid.uidByte[i] < 0x10 ? " 0" : " ");
        readUID += String(rfid.uid.uidByte[i], HEX);
      }
      readUID.toUpperCase();
      readUID = readUID.substring(1); // Strip leading space
      
      Serial.printf("[RFID Scan] Card UID: %s\n", readUID.c_str());
      
      if (readUID == validUID) {
        unlockVault();
        unlockStartTime = millis();
      } else {
        logEvent("REJECTED", "Unauthorized UID scanned: " + readUID);
        playChime(false);
      }
      
      rfid.PICC_HaltA(); // Halt PICC communications
      rfid.PCD_StopCrypto1();
    }
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MFRC522 RFID**, **MPU-6050**, **Servo**, **Buzzer**, and **LED** onto the canvas.
2. Wire the parts to the specified pins.
3. Paste the code and click **Run**.
4. In simulation, since a physical card scan cannot be performed directly, mock an RFID scan by entering `SYS:RFID:de ad be ef` in the serial terminal.
5. Verify that the servo motor widget rotates to 90 degrees, the LED turns ON, and the chime sounds.
6. Verify that the servo rotates back to 0 degrees after 5 seconds.
7. Mock a vibration event by typing `SYS:ACCEL:25.0` in the serial terminal. Verify that the buzzer sounds a warbling siren and access is locked down.
8. Open your browser and navigate to the IP address. Verify that the audit log displays the entry and tamper events. Click "RESET SECURITY LOCKOUT" to clear the alarm.

## Expected Output
Serial Monitor:
```
[MPU-6050] Connected successfully.
[SPIFFS] Mounted successfully.
WiFi Connected.
Local IP Address: 10.10.0.3
[Audit Log] [0] SYSTEM - System armed and active.
[RFID Scan] Card UID: DE AD BE EF
[Audit Log] [12] ACCESS - Vault unlocked via RFID card.
[Audit Log] [17] LOCK - Vault locked automatically.
[Audit Log] [25] TAMPER - Acceleration spike detected: 25.00 m/s^2. LOCKDOWN initiated.
```

## Expected Canvas Behavior
* Triggering simulated RFID or accelerometer events moves the servo motor widget, sounds the buzzer, flashes the LED, and writes the audit logs.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `sqrt(x^2 + y^2 + z^2)` | Mathematical formula calculating the magnitude of the 3D acceleration vector. |
| `rfid.PICC_ReadCardSerial()` | Resolves the card's UID array when a new card is placed on the reader. |
| `SPIFFS.open(..., FILE_APPEND)` | Appends new audit lines to the end of the text log file in flash memory. |

## Hardware & Safety Concept: I2C/SPI Bus Collisions and Mechanical Vault Overrides
* **I2C/SPI Bus Collisions**: The MFRC522 (SPI) and MPU-6050 (I2C) share different hardware buses on the ESP32. Ensure that you do not share physical pins between the I2C SCL/SDA lines and the SPI clock/data lines, which can lock the CPU.
* **Mechanical Vault Overrides**: Relying solely on software and servo motors to lock a door is high-risk: if the battery dies or the ESP32 crashes, the vault door remains locked forever. Always include a physical key bypass or mechanical release latch to open the vault in case of power failure.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display "ARMED - SYSTEM SECURE" or "ALARM! TAMPER DETECTED".
2. **Telegram Alert integration**: Modify the code to send a Telegram push alert (Project 188) when a tamper alarm trips.
3. **Admin authentication**: Protect the web console command routes `/reset` and `/clear` with an HTTP Basic Authentication lock (Project 64).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| RFID reader fails to read card | SPI wiring error | MFRC522 requires correct SPI pins: GPIO 23 (MOSI), 19 (MISO), 18 (SCK), and 5 (SDA/SS). Verify the connections |
| Tamper alarm trips on start | Accelerometer calibration | Gravity exerts a constant ~9.8 m/s^2 downwards. Ensure your calculation compares the total acceleration magnitude against 9.8, not 0 |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [109 - RFID Access Control Lock](../intermediate/109-rfid-access-control-lock.md)
- [124 - Tilt Security Vault](../intermediate/124-tilt-security-vault.md)
- [194 - Greenhouse Automation Hub with IoT Dashboard & Cloud overrides](194-greenhouse-automation-hub-with-iot-dashboard-cloud-overrides.md) (Next project)
