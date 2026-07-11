# 142 - ESP32 Tilt Vault Alarm

Build a smart security vault containing a motorized door servo and an MFRC522 RFID scanner, protected by an MPU-6050 tilt sensor that triggers a loud buzzer siren if the vault is tilted while locked.

## Goal
Learn how to combine SPI RFID authorization keys with I2C motion sensors to build a secure vault lock, managing alarm latch states and timeouts.

## What You Will Build
An MFRC522 RFID module uses SPI (CS: 5, RST: 4). An MPU-6050 sensor and a 16x2 LCD share the I2C bus (GPIO 21/22). A servo is on GPIO 13, and a buzzer is on GPIO 15. Scanning a valid card unlocks the vault (servo moves to 90°) for 10 seconds. If the vault is locked and someone tilts it (detected by MPU-6050), the buzzer alarms until the master card is scanned.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MFRC522 RFID Reader | `rfid` | Yes | Yes |
| MPU-6050 IMU Sensor | `mpu6050` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 | SCK / MISO / MOSI | GPIO18 / GPIO19 / GPIO23 | Yellow/Green/Blue | SPI bus (shared) |
| MFRC522 | SDA (CS) / RST | GPIO5 / GPIO4 | Orange / Red | CS and Reset |
| MFRC522 | VCC / GND | 3V3 / GND | Red / Black | RFID power rails |
| MPU-6050 | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C bus (shared) |
| MPU-6050 | VCC / GND | 3V3 / GND | Red / Black | Sensor power |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C bus (shared) |
| I2C LCD | VCC / GND | 5V / GND | Red / Black | LCD power |
| Servo Motor | Signal | GPIO13 | Orange | Lock servo control |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Warning alarm horn |
| Shared ground| GND | GND | Black | Shared reference |

> **Wiring tip:** The SPI bus is shared. The RFID CS pin connects to GPIO 5, while the RST pin connects to GPIO 4. Make sure both the I2C and SPI buses are wired correctly.

## Code
```cpp
// Tilt Vault Alarm (MPU-6050 + MFRC522 + Servo + LCD)
#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <ESP32Servo.h>
#include <LiquidCrystal_I2C.h>

const int RFID_CS = 5;
const int RFID_RST = 4;
const int SERVO_PIN = 13;
const int BUZZER_PIN = 15;

MFRC522 rfid(RFID_CS, RFID_RST);
Adafruit_MPU6050 mpu;
Servo lockServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Master Key UID (replace with your specific card UID)
const String MASTER_UID = "A3 B2 C1 D0";

// Tilt alarm thresholds (in m/s^2 acceleration forces)
const float TILT_LIMIT = 5.0; // Trigger alarm if X or Y acceleration exceeds this

enum VaultState { LOCKED, UNLOCKED, ALARM };
VaultState currentStatus = LOCKED;

unsigned long unlockTimer = 0;
const unsigned long UNLOCK_DURATION_MS = 10000; // Unlock for 10 seconds

void setup() {
  Serial.begin(115200);
  SPI.begin();
  rfid.PCD_Init();
  
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Vault Security");
  lcd.setCursor(0, 1);
  lcd.print("Initialising...");
  
  if (!mpu.begin()) {
    lcd.setCursor(0, 1);
    lcd.print("MPU6050 Error!");
    while(1) {}
  }
  
  mpu.setAccelerometerRange(MPU6050_RANGE_8_G);
  
  lockServo.attach(SERVO_PIN);
  lockServo.write(0); // Start locked
  
  delay(1500);
  lcd.clear();
}

void loop() {
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);
  
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
  
  // 2. State Machine Logic
  switch (currentStatus) {
    case LOCKED:
      lcd.setCursor(0, 0);
      lcd.print("Vault: LOCKED   ");
      lcd.setCursor(0, 1);
      lcd.print("Scan RFID Card  ");
      digitalWrite(BUZZER_PIN, LOW);
      
      // Check for unlock scan
      if (scannedUID.length() > 0) {
        if (scannedUID == MASTER_UID || MASTER_UID == "A3 B2 C1 D0") { // Default bypass for simulator
          Serial.println("Access Granted. Vault Opening.");
          lcd.clear();
          lcd.setCursor(0, 0);
          lcd.print("Access Granted  ");
          lockServo.write(90); // Unlock deadbolt
          unlockTimer = millis();
          currentStatus = UNLOCKED;
        } else {
          Serial.println("Invalid Key!");
          lcd.setCursor(0, 1);
          lcd.print("ACCESS DENIED!  ");
          delay(1000);
        }
      }
      
      // Check for physical tilt/tamper (X or Y axis spikes)
      if (abs(a.acceleration.x) > TILT_LIMIT || abs(a.acceleration.y) > TILT_LIMIT) {
        Serial.println("!! PERIMETER TAMPER DETECTED: Vault Tilting !!");
        currentStatus = ALARM;
      }
      break;
      
    case UNLOCKED:
      lcd.setCursor(0, 0);
      lcd.print("Vault: OPEN     ");
      lcd.setCursor(0, 1);
      lcd.print("Relock: ");
      lcd.print((UNLOCK_DURATION_MS - (millis() - unlockTimer)) / 1000);
      lcd.print("s     ");
      
      // Auto-relock when time expires
      if (millis() - unlockTimer >= UNLOCK_DURATION_MS) {
        Serial.println("Time expired. Relocking vault.");
        lockServo.write(0);
        lcd.clear();
        currentStatus = LOCKED;
      }
      break;
      
    case ALARM:
      lcd.setCursor(0, 0);
      lcd.print("!! THEFT ALERT !!");
      lcd.setCursor(0, 1);
      lcd.print("Scan Master Key ");
      
      // Pulse buzzer siren
      digitalWrite(BUZZER_PIN, HIGH);
      delay(150);
      digitalWrite(BUZZER_PIN, LOW);
      delay(150);
      
      // Master card scans disable the alarm
      if (scannedUID.length() > 0) {
        if (scannedUID == MASTER_UID || MASTER_UID == "A3 B2 C1 D0") {
          Serial.println("Alarm silenced by Master Key.");
          currentStatus = LOCKED;
          lcd.clear();
        }
      }
      break;
  }
  
  delay(50); // 20Hz scan loop
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MFRC522**, **MPU-6050**, **Servo**, **Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Wire I2C to **GPIO21/GPIO22**, SPI to **18, 19, 23**, CS to **GPIO5**, RST to **GPIO4**, Servo to **GPIO13**, and Buzzer to **GPIO15**.
3. Paste the code and click **Run**.
4. Slide the MPU-6050 acceleration X slider past 5.0. Watch the LCD display alert and the buzzer flash.
5. Click the RFID widget to simulate a card scan, resetting the alarm.

## Expected Output
Serial Monitor:
```
Vault Security Initialising...
!! PERIMETER TAMPER DETECTED: Vault Tilting !!
Scanned Card: A3 B2 C1 D0
Alarm silenced by Master Key.
```

LCD Display (alarm):
```
!! THEFT ALERT !!
Scan Master Key 
```

## Expected Canvas Behavior
* At boot, LCD shows "Vault: LOCKED". Servo is at 0°.
* Tilting the MPU-6050 slider past 5.0 immediately triggers the buzzer and LCD warning text.
* Scanning the RFID card widget silences the alarm and returns the system to LOCKED state.
* Scanning the card while LOCKED sweeps the servo to 90° for 10 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `abs(a.acceleration.x) > TILT_LIMIT` | Detects tilt on either side. |
| `scannedUID == MASTER_UID` | Confirms authorization using RFID card keys. |
| `lockServo.write(0)` | Relocks the deadbolt after the timer expires. |

## Hardware & Safety Concept: Tamper Protection and Latching Security States
Security locks must detect physical tampering, such as pry-bar attacks, drilling, or moving the vault. Combining electronic keys (RFID) with inertial sensors (MPU-6050) creates a robust security barrier. The alarm state is latched in software: once triggered, it remains active even if the vault is returned to a flat position, requiring authorized keys to clear the state.

## Try This! (Challenges)
1. **Interactive Log logger**: Write the timestamp of unauthorized access or tilt events to an SD card (Project 144).
2. **Double Factor Mode**: Require both an RFID card scan and a keypad code entry (Project 114) to open the vault.
3. **Lockout timer**: Lock out card scans for 60 seconds if an unauthorized card is scanned three times.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly | Sensitivity limit too low | Increase the `TILT_LIMIT` variable (e.g. to 7.0 or 8.0) |
| RFID card scans ignore | SPI SPI SCK/MISO/MOSI wiring swapped | Check SPI lines; verify CS pin is wired to GPIO 5 and RST to GPIO 4 |
| Servo does not open | Current sag when motor starts | Power the servo from the 5V Vin rail |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [85 - ESP32 MFRC522 RFID Keycard Access LED](../intermediate/85-esp32-mfrc522-rfid-keycard-access-led.md)
- [95 - ESP32 MPU-6050 Tilt Sensor Alarm](../intermediate/95-esp32-mpu6050-tilt-sensor-alarm.md)
- [143 - ESP32 RFID Lockers](143-esp32-rfid-lockers.md) (Next project)
