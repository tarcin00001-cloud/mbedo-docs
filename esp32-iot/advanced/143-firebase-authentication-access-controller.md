# 143 - Firebase Authentication Access Controller

Build a cloud-authorized smart lock access controller on the ESP32 that reads an MFRC522 RFID reader over SPI, sends an HTTPS request to a Firebase database endpoint to verify user authorization, and controls a lock servo on GPIO 12.

## Goal
Learn how to interface RFID sensors over SPI, perform HTTP query lookups on cloud databases, parse JSON responses, control servo motors, and log access attempts.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An MFRC522 RFID reader is on SPI, and a servo on GPIO 12 simulates a door latch. Swiping an RFID card reads its unique identifier (UID). The ESP32 queries a Firebase NoSQL database endpoint (`https://esp32-default-rtdb.firebaseio.com/users/[UID].json`) using an HTTP GET request. If the database returns `{"authorized":true}`, the servo rotates to 90° to unlock; if unauthorized, the lock remains closed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MFRC522 RFID Reader | `rfid` | Yes | Yes |
| RFID Cards / Key fobs | `rfid` | Yes (2 pieces) | Yes (2 pieces) |
| Servo Motor | `servo` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, click the RFID sensor widget to select which card UID to swipe.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| RFID Module | SDA (SS) | GPIO5 | Yellow | SPI chip select |
| RFID Module | SCK | GPIO18 | Green | SPI clock |
| RFID Module | MISO | GPIO19 | Blue | SPI master input |
| RFID Module | MOSI | GPIO23 | White | SPI master output |
| RFID Module | RST | GPIO14 | Orange | Reset signal input |
| Servo Motor | PWM (Signal) | GPIO12 | Red | Lock actuator signal |
| Buzzer | Positive (+) | GPIO13 | Purple | Alarm audio chime |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the RFID reader from the 3.3V rail and the servo motor from the 5V Vin rail.

## Code
```cpp
// Firebase Authentication Access Controller (RFID SPI interface + Firebase Auth check)
#include <WiFi.h>
#include <HTTPClient.h>
#include <SPI.h>
#include <MFRC522.h>
#include <ESP32Servo.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Firebase REST Endpoint URL for looking up card authorization status
// Format: https://[project-id].firebaseio.com/users/[cardUID].json
const String firebaseBaseUrl = "https://esp32-default-rtdb.firebaseio.com/users/";

const int RFID_SS_PIN = 5;
const int RFID_RST_PIN = 14;
MFRC522 mfrc522(RFID_SS_PIN, RFID_RST_PIN);

const int SERVO_PIN = 12;
const int BUZZER_PIN = 13;
Servo lockServo;

// Local fallback user list (if database connection is offline or in simulation)
const String LOCAL_USER_UID = "4A8F9C2D";

void soundChime(bool granted) {
  if (granted) {
    tone(BUZZER_PIN, 1000);
    delay(100);
    tone(BUZZER_PIN, 1500);
    delay(150);
    noTone(BUZZER_PIN);
  } else {
    tone(BUZZER_PIN, 400);
    delay(400);
    noTone(BUZZER_PIN);
  }
}

// Queries Firebase Database REST API to check authorization status
bool checkAuthorization(String uid) {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    String url = firebaseBaseUrl + uid + ".json";
    
    Serial.printf("[Cloud Auth] Querying Firebase node: %s\n", url.c_str());
    http.begin(url);
    
    int httpResponseCode = http.GET();
    bool authorized = false;
    
    if (httpResponseCode > 0) {
      String response = http.getString();
      Serial.printf("[Cloud Auth] Response: %s\n", response.c_str());
      
      // Basic JSON parsing to search for authorized field
      if (response.indexOf("\"authorized\":true") != -1) {
        authorized = true;
      }
    } else {
      Serial.printf("[Cloud Auth] Connection error: %d. Relying on local lookup.\n", httpResponseCode);
      // Fallback local lookup if server is not configured or offline
      if (uid == LOCAL_USER_UID) {
        authorized = true;
      }
    }
    http.end();
    return authorized;
  } else {
    Serial.println("[Cloud Auth] WiFi Offline. Relying on local lookup.");
    return (uid == LOCAL_USER_UID);
  }
}

void openLock() {
  Serial.println("[Lock] Access Granted! Unlocking door.");
  soundChime(true);
  lockServo.write(90); // Rotate servo to 90 degrees to unlock
  delay(3000);        // Hold door open for 3 seconds
  lockServo.write(0);  // Rotate back to 0 degrees to lock
  Serial.println("[Lock] Relocking door.");
}

void denyLock() {
  Serial.println("[Lock] Access Denied! Lock remains closed.");
  soundChime(false);
  lockServo.write(0);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Initialize SPI Bus and RFID Reader
  SPI.begin();
  mfrc522.PCD_Init();
  
  lockServo.attach(SERVO_PIN);
  lockServo.write(0); // Set initial lock state to closed
  
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("\nESP32 RFID Lock starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  Serial.println("Scan an RFID card to verify authorization.");
}

void loop() {
  // Check for new RFID cards
  if (!mfrc522.PICC_IsNewCardPresent()) {
    return;
  }
  if (!mfrc522.PICC_ReadCardSerial()) {
    return;
  }
  
  // Read Card UID and convert to Hex string
  String cardUid = "";
  for (byte i = 0; i < mfrc522.uid.size; i++) {
    cardUid += String(mfrc522.uid.uidByte[i] < 0x10 ? "0" : "");
    cardUid += String(mfrc522.uid.uidByte[i], HEX);
  }
  cardUid.toUpperCase();
  
  Serial.printf("\n[RFID] Scanned card UID: %s\n", cardUid.c_str());
  
  // Perform cloud verification
  bool isAuthorized = checkAuthorization(cardUid);
  
  if (isAuthorized) {
    openLock();
  } else {
    denyLock();
  }
  
  // Halt PICC
  mfrc522.PICC_HaltA();
  delay(1000); // 1-second delay between scans
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MFRC522 RFID Module**, **Servo Motor**, and **Buzzer** onto the canvas.
2. Wire RFID to SPI pins (CS to **GPIO5**, SCK to **GPIO18**, MISO to **GPIO19**, MOSI to **GPIO23**, RST to **GPIO14**).
3. Wire Servo to **GPIO12** and Buzzer to **GPIO13**.
4. Paste the code and click **Run**.
5. Click on the RFID widget, select the card with UID `4A8F9C2D` (the local fallback), and click swipe.
6. Verify that the buzzer plays a success chime, the servo rotates to 90 degrees, and relocks after 3 seconds.
7. Swipe an unauthorized card. Verify that the buzzer plays a low error tone and the servo stays at 0 degrees.

## Expected Output
Serial Monitor:
```
WiFi Connected.
ESP32 RFID Lock starting...
Scan an RFID card to verify authorization.

[RFID] Scanned card UID: 4A8F9C2D
[Cloud Auth] Querying Firebase node: https://esp32-default-rtdb.firebaseio.com/users/4A8F9C2D.json
[Cloud Auth] Connection error: -1. Relying on local lookup.
[Lock] Access Granted! Unlocking door.
[Lock] Relocking door.
```

## Expected Canvas Behavior
* Swiping the authorized card rotates the servo widget to 90 degrees and sounds the success chime.
* Swiping an unauthorized card triggers a warning tone and keeps the servo locked.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `mfrc522.PICC_IsNewCardPresent()` | Checks if a new card is placed near the reader. |
| `cardUid += String(..., HEX)` | Formats the raw binary card bytes into a hexadecimal string. |
| `firebaseBaseUrl + uid + ".json"` | Constructs the Firebase database query path for the card. |
| `indexOf("\"authorized\":true")` | Checks if the JSON response indicates success. |

## Hardware & Safety Concept: Local Access Fallback and Servo Power Isolation
* **Local Fallback**: In cloud-managed lock systems, if WiFi drops or the database server goes offline, users could be locked out. Implementing a local list of master keys (like `LOCAL_USER_UID`) directly in the ESP32 code ensures essential staff can still unlock the door during network outages.
* **Power Isolation**: High-torque lock solenoids or servos draw large currents. If powered from the ESP32 5V pin, they can cause voltage drops that reset the board. Always power the lock actuator from a separate power supply and share the ground pin.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display "Access Granted" or "Access Denied" animations.
2. **Access Log Webpage**: Host a local web page on the ESP32 showing a table of recent entry times and swiped card UIDs.
3. **Database Write on swipe**: Program the ESP32 to upload a log entry to Firebase showing the timestamp of the access attempt.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| RFID reader fails to read cards | Loose SPI connections | Check your wiring. Ensure SCK, MISO, MOSI, and SS are mapped correctly |
| Card reads but server rejects | JSON mismatch | Check the database node payload. Ensure it contains the exact string `"authorized":true` |
| Servo vibrates instead of rotating | Low power | Ensure the servo is powered from a stable external 5V supply |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [114 - ESP32 Keypad 4x4 password lock with access logs](114-esp32-keypad-4x4-password-lock-iot-keypad-servo-web-access-logs.md)
- [142 - Firebase Realtime Database Logger](142-firebase-realtime-database-logger-upload-current-state-to-firebase.md)
- [144 - Adafruit IO Dashboard logger](144-adafruit-io-dashboard-logger-push-values-to-adafruit-io-dashboard.md) (Next project)
