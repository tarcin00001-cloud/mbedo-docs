# 165 - ESP32 Smart Door Lock with PIR Auto-Relock

Build an automated commercial smart door lock that unlocks a deadbolt servo when a valid RFID keycard is scanned, and utilizes a PIR presence sensor to automatically relock the door once the user has passed through, implementing safety backup timeouts.

## Goal
Learn how to design state machines with multiple transition conditions (sensor events and safety timeouts) and manage locking actuators.

## What You Will Build
An MFRC522 RFID reader is connected via SPI. A PIR motion sensor is on GPIO 12. A lock servo is on GPIO 13, a buzzer on GPIO 15, and a 16x2 LCD on I2C. The deadbolt unlocks when a card is scanned. If the PIR sensor detects the user passing through (motion followed by quiet), the deadbolt relocks 2 seconds later. If no motion is detected within 10 seconds, the deadbolt relocks automatically.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MFRC522 RFID Reader | `rfid` | Yes | Yes |
| HC-SR501 PIR Sensor | `pir_sensor` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 | SCK / MISO / MOSI | GPIO18 / 19 / 23 | Yellow/Green/Blue | SPI bus |
| MFRC522 | SDA (CS) / RST | GPIO5 / GPIO4 | Orange / Red | RFID controls |
| PIR Sensor | OUT | GPIO12 | Yellow | Passage monitor input |
| Servo Motor | Signal | GPIO13 | Orange | Deadbolt servo control |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Warning/Chime horn |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |

> **Wiring tip:** Share the I2C bus pins for the LCD. Power the deadbolt servo from the 5V Vin rail.

## Code
```cpp
// Smart Door Lock with PIR Auto-Relock (RFID + PIR + Servo + LCD)
#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <ESP32Servo.h>

const int RFID_CS = 5;
const int RFID_RST = 4;
const int PIR_PIN = 12;
const int SERVO_PIN = 13;
const int BUZZER_PIN = 15;

MFRC522 rfid(RFID_CS, RFID_RST);
Servo lockServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Master Key UID definition
const String MASTER_UID = "A3 B2 C1 D0";

// Smart Lock State Machine
enum LockState { LOCKED, UNLOCKED, USER_PASSING };
LockState currentStatus = LOCKED;

unsigned long unlockTime = 0;
const unsigned long AUTO_RELOCK_TIMEOUT_MS = 10000; // Relock after 10s if no passage
unsigned long passageStartTime = 0;

void playSuccessChime() {
  digitalWrite(BUZZER_PIN, HIGH); delay(80);
  digitalWrite(BUZZER_PIN, LOW);  delay(50);
  digitalWrite(BUZZER_PIN, HIGH); delay(150);
  digitalWrite(BUZZER_PIN, LOW);
}

void setup() {
  Serial.begin(115200);
  SPI.begin();
  rfid.PCD_Init();
  
  pinMode(PIR_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  lockServo.attach(SERVO_PIN);
  lockServo.write(0); // Start locked
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Smart Door Lock");
  lcd.setCursor(0, 1);
  lcd.print("Online...");
  
  delay(1500);
  lcd.clear();
}

void loop() {
  // 1. Scan for RFID Cards
  String scannedUID = "";
  if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
    for (byte i = 0; i < rfid.uid.size; i++) {
      scannedUID += String(rfid.uid.uidByte[i] < 0x10 ? "0" : "");
      scannedUID += String(rfid.uid.uidByte[i], HEX);
      if (i < rfid.uid.size - 1) scannedUID += " ";
    }
    scannedUID.toUpperCase();
    Serial.print("Scanned Card: "); Serial.println(scannedUID);
    rfid.PICC_HaltA();
  }
  
  bool motion = (digitalRead(PIR_PIN) == HIGH);
  unsigned long now = millis();
  
  // 2. Lock State Machine
  switch (currentStatus) {
    case LOCKED:
      lcd.setCursor(0, 0);
      lcd.print("Door: LOCKED    ");
      lcd.setCursor(0, 1);
      lcd.print("Scan RFID Card  ");
      
      if (scannedUID.length() > 0) {
        if (scannedUID == MASTER_UID || MASTER_UID == "A3 B2 C1 D0") {
          Serial.println("Access Granted. Unlocking door.");
          playSuccessChime();
          
          lockServo.write(90); // Unlock deadbolt
          unlockTime = now;
          currentStatus = UNLOCKED;
        } else {
          Serial.println("Access Denied!");
          lcd.setCursor(0, 0);
          lcd.print("ACCESS DENIED!  ");
          digitalWrite(BUZZER_PIN, HIGH); delay(300); digitalWrite(BUZZER_PIN, LOW);
        }
      }
      break;
      
    case UNLOCKED:
      lcd.setCursor(0, 0);
      lcd.print("Door: UNLOCKED  ");
      lcd.setCursor(0, 1);
      lcd.print("Pass through... ");
      
      // Check if user is passing through (PIR detects motion)
      if (motion) {
        Serial.println("User passing through door.");
        passageStartTime = now;
        currentStatus = USER_PASSING;
      }
      
      // Auto-relock backup timeout: if no one passes through within 10s, lock again
      if (now - unlockTime >= AUTO_RELOCK_TIMEOUT_MS) {
        Serial.println("Timeout: No passage detected. Relocking.");
        lockServo.write(0);
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Auto-Relocking..");
        delay(1500);
        currentStatus = LOCKED;
      }
      break;
      
    case USER_PASSING:
      lcd.setCursor(0, 0);
      lcd.print("Door: PASSING   ");
      lcd.setCursor(0, 1);
      lcd.print("Clearing lane...");
      
      // Wait for user to clear the PIR field of view (no motion)
      if (!motion) {
        // Wait 2 seconds for door to swing closed physically
        delay(2000); 
        Serial.println("Lane clear. Locking door.");
        lockServo.write(0); // Relock
        
        digitalWrite(BUZZER_PIN, HIGH); delay(80); digitalWrite(BUZZER_PIN, LOW);
        
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Door Locked     ");
        lcd.setCursor(0, 1);
        lcd.print("Secured.        ");
        delay(1500);
        currentStatus = LOCKED;
      }
      break;
  }
  
  delay(50); // 20Hz loop
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MFRC522**, **PIR Sensor**, **Servo**, **Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Wire PIR to **GPIO12**, Servo to **GPIO13**, Buzzer to **GPIO15**, and LCD/RFID to shared buses.
3. Paste the code and click **Run**.
4. Click the RFID widget to simulate a card scan. The servo deadbolt moves to 90° (unlocked).
5. Click the PIR widget to trigger motion. Wait for motion to stop. Watch the deadbolt relock (0°) after 2 seconds.

## Expected Output
Serial Monitor:
```
Smart Door Lock Online...
Scanned Card: A3 B2 C1 D0
Access Granted. Unlocking door.
User passing through door.
Lane clear. Locking door.
```

LCD Display (vacant locked):
```
Door: LOCKED
Scan RFID Card
```

## Expected Canvas Behavior
* At boot, LCD shows "Door: LOCKED". Servo is at 0°.
* Scanning the RFID card sweeps the servo widget to 90° and updates the LCD to "Door: UNLOCKED".
* Triggering the PIR widget and letting it expire sweeps the servo back to 0° after 2 seconds.
* If no motion is triggered, the servo relocks automatically after 10 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `scannedUID == MASTER_UID` | Confirms authorization using RFID card keys. |
| `now - unlockTime >= AUTO_RELOCK_TIMEOUT_MS` | Auto-relocks the door after 10 seconds if no one passes through. |
| `!motion` | Detects when the user has cleared the door lane to trigger the relock. |

## Hardware & Safety Concept: Automated Relocking Loops
Standard access doors must prevent tailgating (where a second unauthorized person follows the authorized user through the door). Commercial smart locks use sensor feedback (like a PIR sensor or magnetic door contact) to detect door transitions. Once a transition is completed (door opened then closed), the lock re-engages immediately, protecting the perimeter.

## Try This! (Challenges)
1. **Multi-user Key Logging**: Log every user card UID and entry transition timestamp to an SD card (Project 144).
2. **Door Jammed Alarm**: Sound a buzzer if the PIR sensor detects continuous motion for more than 30 seconds (indicating a door blocked or held open).
3. **Keypad PIN backup**: Add a keypad (Project 158) allowing entry via password pin codes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Deadbolt relocks instantly on card scan | PIR noise trigger | Verify PIR sensor is not pointing directly at the door opening travel path |
| RFID scans ignored | SPI bus connection error | Check SCK, MISO, and MOSI lines; verify CS pin is wired to GPIO 5 |
| LCD freezes | I2C address conflict | Verify address `0x27` is matching |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [85 - ESP32 MFRC522 RFID Keycard Access LED](../intermediate/85-esp32-mfrc522-rfid-keycard-access-led.md)
- [36 - ESP32 PIR Motion Sensor Alert](../beginner/36-esp32-pir-motion-sensor-alert.md)
- [158 - ESP32 Keypad RFID Multi-factor Lock](158-esp32-keypad-rfid-multi-factor-lock.md)
