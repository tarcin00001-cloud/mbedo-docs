# 143 - ESP32 RFID Lockers

Build a temporary public locker storage lock that allows any user to scan an RFID card to lock an open cabinet, registering that card's unique UID as the key, and requiring the same card to unlock it.

## Goal
Learn how to implement dynamic key registration, compare runtime string variables, manage state machines, and actuate locking servo deadbolts.

## What You Will Build
An MFRC522 RFID reader uses SPI (CS: 5, RST: 4). A 16x2 LCD is on I2C. A servo (lock) is on GPIO 13, and a buzzer is on GPIO 15. The locker starts OPEN (servo at 90°). When a user scans an RFID card, the system saves its UID as the key, locks the cabinet (servo at 0°), and displays "LOCKED". Scanning the same card unlocks the cabinet and clears the key; scanning any other card triggers a warning.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MFRC522 RFID Reader | `rfid` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 | SCK / MISO / MOSI | GPIO18 / GPIO19 / GPIO23 | Yellow/Green/Blue | SPI bus |
| MFRC522 | SDA (CS) / RST | GPIO5 / GPIO4 | Orange / Red | CS and Reset |
| MFRC522 | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| I2C LCD | VCC / GND | 5V / GND | Red / Black | LCD power |
| Servo Motor | Signal | GPIO13 | Orange | Lock servo control |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Alarm buzzer |

> **Wiring tip:** Share the I2C bus pins in parallel (GPIO 21 and 22) for the LCD. Power the lock servo from the 5V Vin rail.

## Code
```cpp
// RFID Lockers (MFRC522 + LCD + Servo + Buzzer)
#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <ESP32Servo.h>

const int RFID_CS = 5;
const int RFID_RST = 4;
const int SERVO_PIN = 13;
const int BUZZER_PIN = 15;

MFRC522 rfid(RFID_CS, RFID_RST);
Servo lockServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

enum LockerState { OPEN, LOCKED };
LockerState currentStatus = OPEN;

String registeredOwnerUID = ""; // Stores the temporary key UID

void playSuccessChime() {
  digitalWrite(BUZZER_PIN, HIGH); delay(100);
  digitalWrite(BUZZER_PIN, LOW);  delay(80);
  digitalWrite(BUZZER_PIN, HIGH); delay(200);
  digitalWrite(BUZZER_PIN, LOW);
}

void playErrorChime() {
  digitalWrite(BUZZER_PIN, HIGH); delay(300);
  digitalWrite(BUZZER_PIN, LOW);
}

void setup() {
  Serial.begin(115200);
  SPI.begin();
  rfid.PCD_Init();
  
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  lockServo.attach(SERVO_PIN);
  lockServo.write(90); // Start OPEN
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("RFID Locker Sys");
  lcd.setCursor(0, 1);
  lcd.print("Initialising...");
  
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
    Serial.print("Scanned: "); Serial.println(scannedUID);
    rfid.PICC_HaltA();
  }
  
  // 2. Locker Logic
  if (scannedUID.length() > 0) {
    if (currentStatus == OPEN) {
      // Lock and save this card as the owner key
      registeredOwnerUID = scannedUID;
      lockServo.write(0); // Sweep to lock position
      currentStatus = LOCKED;
      
      playSuccessChime();
      
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("LOCKER LOCKED!");
      lcd.setCursor(0, 1);
      lcd.print("Key registered");
      delay(2000);
      lcd.clear();
    } 
    else if (currentStatus == LOCKED) {
      // Check if scanned card matches the owner key
      if (scannedUID == registeredOwnerUID) {
        // Unlock and reset
        registeredOwnerUID = "";
        lockServo.write(90); // Sweep to unlock position
        currentStatus = OPEN;
        
        playSuccessChime();
        
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("UNLOCKED!");
        lcd.setCursor(0, 1);
        lcd.print("Locker open.");
        delay(2000);
        lcd.clear();
      } else {
        // Wrong key
        playErrorChime();
        
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("WRONG CARD!");
        lcd.setCursor(0, 1);
        lcd.print("Access Denied");
        delay(2000);
        lcd.clear();
      }
    }
  }
  
  // Display idle HUD
  lcd.setCursor(0, 0);
  if (currentStatus == OPEN) {
    lcd.print("Locker: VACANT  ");
    lcd.setCursor(0, 1);
    lcd.print("Scan card to lock");
  } else {
    lcd.print("Locker: OCCUPIED");
    lcd.setCursor(0, 1);
    lcd.print("Scan key to open");
  }
  
  delay(100);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MFRC522**, **Servo**, **Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Wire shared SPI/I2C buses, CS to **GPIO5**, RST to **GPIO4**, Servo to **GPIO13**, and Buzzer to **GPIO15**.
3. Paste the code and click **Run**.
4. Observe "Locker: VACANT" on the LCD. Click the RFID widget to simulate a card scan. The servo deadbolt moves to 0° (locked).
5. Click a *different* card simulation or rescan to unlock.

## Expected Output
Serial Monitor:
```
RFID Locker Sys Initialising...
Scanned: A3 B2 C1 D0
Locker Locked. Key registered.
Scanned: B4 C5 D6 E7
Access Denied: Wrong Card!
Scanned: A3 B2 C1 D0
Locker Unlocked.
```

LCD Display (occupied):
```
Locker: OCCUPIED
Scan key to open
```

## Expected Canvas Behavior
* At boot, the LCD displays "Locker: VACANT". Servo is at 90°.
* Clicking the RFID card widget locks the servo to 0° and updates the LCD to "Locker: OCCUPIED".
* Clicking a card with a mismatched UID triggers the buzzer error tone and shows "WRONG CARD!".
* Rescanning the matching card unlocks the servo and resets the LCD to vacant.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `registeredOwnerUID = scannedUID` | Registers the current scanned card's UID as the temporary locker key. |
| `scannedUID == registeredOwnerUID` | Compares UIDs to authorize unlocking. |
| `registeredOwnerUID = ""` | Resets the key buffer to vacant. |

## Hardware & Safety Concept: Dynamic Registrations and Public Safety Locks
Standard hotel safes and public lockers utilize **dynamic key registration**. Instead of having a hardcoded key, the safe is left unlocked. The guest places their items inside, types a code (or scans their room card), and the safe locks, registering that transient code/UID as the key. This ensures each guest has unique access, clearing the memory upon checkout so the locker is vacant for the next user.

## Try This! (Challenges)
1. **Master Override Key**: Hardcode a master administrator key UID that can bypass and unlock the locker in an emergency.
2. **Occupancy Timer**: Display how long the locker has been occupied on Row 2 in minutes and seconds.
3. **Breach Alarm Siren**: Add a vibration sensor (Project 106). If vibration is detected while occupied, trigger a theft alarm.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Cabinet locks but refuses to unlock with same card | Card reading corruption | Clean the sensor window; print scanned UIDs to Serial to verify matches |
| LCD flashes when servo operates | Motor drawing logic current | Connect the servo VCC wire to Vin |
| Card scans are ignored | SPI bus connection error | Verify MISO, MOSI, and SCK are wired correctly |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [85 - ESP32 MFRC522 RFID Keycard Access LED](../intermediate/85-esp32-mfrc522-rfid-keycard-access-led.md)
- [142 - ESP32 Tilt Vault Alarm](142-esp32-tilt-vault-alarm.md)
- [144 - ESP32 RFID Access Logger](144-esp32-rfid-access-logger.md) (Next project)
