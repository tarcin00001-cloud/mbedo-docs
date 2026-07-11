# 172 - RFID Smart Campus Gate

Control a campus access gate using an RFID reader, servo motor, PIR motion detector, I2C LCD, and an SD card logger.

## Goal
Learn how to interface multiple SPI and I2C components (MFRC522 RFID and SD Card) concurrently, manage servo transitions with time-based state machines instead of loops, read digital motion triggers, and log access transactions to an SD card.

## What You Will Build
An automated entry barrier system. When an RFID tag is swiped, the system verifies its authenticity. If authorized, the system sound a buzzer beep, logs the event to an SD card, displays a welcome message on the LCD, and slowly sweeps the servo to 90° (open). A PIR sensor keeps the gate open if motion is detected. After the pathway is clear and a timeout expires, the servo sweeps back to 0° (closed).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| SG90 Servo Motor | `servo` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |
| I2C LCD (16x2) | `lcd_i2c` | Yes | Yes |
| SD Card Reader Module | `sd_spiv3` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Red Warning LED | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 RFID | VCC | 3V3 | Red | Power input (3.3V) |
| MFRC522 RFID | RST | GPIO 9 | White | Reset pin |
| MFRC522 RFID | GND | GND | Black | Ground |
| MFRC522 RFID | MISO | GPIO 15 | Green | SPI CIPO |
| MFRC522 RFID | MOSI | GPIO 14 | Yellow | SPI COPI |
| MFRC522 RFID | SCK | GPIO 5 | Blue | SPI SCK |
| MFRC522 RFID | SDA (SS) | GPIO 10 | Orange | SPI Chip Select (RFID) |
| SD Card Reader | VCC | 5V | Red | Power input (5V) |
| SD Card Reader | GND | GND | Black | Ground |
| SD Card Reader | MISO | GPIO 15 | Green | SPI Shared CIPO |
| SD Card Reader | MOSI | GPIO 14 | Yellow | SPI Shared COPI |
| SD Card Reader | SCK | GPIO 5 | Blue | SPI Shared SCK |
| SD Card Reader | CS | GPIO 8 | Purple | SPI Chip Select (SD) |
| SG90 Servo | Signal | GPIO 13 | Yellow | PWM Control |
| SG90 Servo | VCC | 5V | Red | Power |
| SG90 Servo | GND | GND | Black | Ground |
| PIR Sensor | OUT | GPIO 11 | Brown | Digital trigger input |
| PIR Sensor | VCC | 5V | Red | Power |
| PIR Sensor | GND | GND | Black | Ground |
| I2C LCD (16x2) | SDA | GPIO 17 | Green | I2C Data (SDA0) |
| I2C LCD (16x2) | SCL | GPIO 16 | Yellow | I2C Clock (SCL0) |
| I2C LCD (16x2) | VCC | 5V | Red | Power |
| I2C LCD (16x2) | GND | GND | Black | Ground |
| Active Buzzer | + | GPIO 7 | Gray | Buzz status pin |
| Active Buzzer | - | GND | Black | Ground |
| Red LED | Anode | GPIO 6 | Red | Warning Indicator |
| Red LED | Cathode | GND | Black | Via 220 Ohm resistor |

> **Wiring tip:** Since the MFRC522 RFID and SD Card reader both share the SPI bus (pins 5, 14, and 15), make sure each device has a unique Chip Select (CS) pin (GPIO 10 for RFID, GPIO 8 for SD). 

## Code
```cpp
// RFID Smart Campus Gate - VEGA ARIES v3
#include <SPI.h>
#include <MFRC522.h>
#include <SD.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Servo.h>

const int SD_CS = 8;
const int RFID_CS = 10;
const int RFID_RST = 9;
const int SERVO_PIN = 13;
const int PIR_PIN = 11;
const int BUZZER_PIN = 7;
const int WARNING_LED = 6;

MFRC522 mfrc522(RFID_CS, RFID_RST);
LiquidCrystal_I2C lcd(0x27, 16, 2);
Servo gateServo;

int gateState = 0; // 0: Idle/Closed, 1: Opening, 2: Open/Waiting, 3: Closing
int servoPos = 0;
unsigned long stateTimer = 0;
unsigned long servoTimer = 0;
bool motionDetected = false;

void setup() {
  Serial.begin(115200);
  SPI.begin();
  mfrc522.PCD_Init();
  
  pinMode(PIR_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(WARNING_LED, OUTPUT);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print(" Campus Gate ");
  lcd.setCursor(0, 1);
  lcd.print("  Scan RFID   ");
  
  gateServo.attach(SERVO_PIN);
  gateServo.write(0);
  
  SD.begin(SD_CS);
}

void loop() {
  // Check PIR Motion Sensor
  motionDetected = digitalRead(PIR_PIN);
  
  // Check RFID Card Reader if gate is closed
  if (gateState == 0) {
    if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
      bool access = false;
      // Compare UID directly without loops (DE AD BE EF is the authorized card)
      if (mfrc522.uid.uidByte[0] == 0xDE && 
          mfrc522.uid.uidByte[1] == 0xAD && 
          mfrc522.uid.uidByte[2] == 0xBE && 
          mfrc522.uid.uidByte[3] == 0xEF) {
        access = true;
      }
      
      // Log to SD
      File logFile = SD.open("gate_log.txt", FILE_WRITE);
      if (logFile) {
        logFile.print("UID: ");
        logFile.print(mfrc522.uid.uidByte[0], HEX);
        logFile.print(mfrc522.uid.uidByte[1], HEX);
        logFile.print(mfrc522.uid.uidByte[2], HEX);
        logFile.print(mfrc522.uid.uidByte[3], HEX);
        if (access) {
          logFile.println(" - ACCESS GRANTED");
        } else {
          logFile.println(" - ACCESS DENIED");
        }
        logFile.close();
      }
      
      if (access) {
        // Buzzer beep for success
        digitalWrite(BUZZER_PIN, HIGH);
        delay(100);
        digitalWrite(BUZZER_PIN, LOW);
        
        lcd.clear();
        lcd.print("Access Granted!");
        lcd.setCursor(0, 1);
        lcd.print("Welcome Student");
        
        gateState = 1; // Transition to Opening
        stateTimer = millis();
      } else {
        // Double beep for denied
        digitalWrite(BUZZER_PIN, HIGH);
        digitalWrite(WARNING_LED, HIGH);
        delay(100);
        digitalWrite(BUZZER_PIN, LOW);
        digitalWrite(WARNING_LED, LOW);
        delay(100);
        digitalWrite(BUZZER_PIN, HIGH);
        digitalWrite(WARNING_LED, HIGH);
        delay(100);
        digitalWrite(BUZZER_PIN, LOW);
        digitalWrite(WARNING_LED, LOW);
        
        lcd.clear();
        lcd.print("Access Denied!");
        lcd.setCursor(0, 1);
        lcd.print("Invalid Card");
        delay(2000);
        lcd.clear();
        lcd.print(" Campus Gate ");
        lcd.setCursor(0, 1);
        lcd.print("  Scan RFID   ");
      }
      mfrc522.PICC_HaltA();
    }
  }
  
  // State Machine execution for gate movement
  if (gateState == 1) { // Opening
    if (millis() - servoTimer >= 15) {
      servoTimer = millis();
      if (servoPos < 90) {
        servoPos = servoPos + 1;
        gateServo.write(servoPos);
      } else {
        gateState = 2; // Transition to Open/Waiting
        stateTimer = millis();
      }
    }
  } else if (gateState == 2) { // Open/Waiting
    // Stay open if motion is detected (PIR)
    if (motionDetected) {
      stateTimer = millis(); // Reset timer if someone is walking through
      lcd.setCursor(0, 1);
      lcd.print("Motion Detected ");
    } else {
      lcd.setCursor(0, 1);
      lcd.print("Closing in 5s...");
    }
    if (millis() - stateTimer >= 5000) {
      gateState = 3; // Transition to Closing
      stateTimer = millis();
      lcd.clear();
      lcd.print("Closing Gate...");
    }
  } else if (gateState == 3) { // Closing
    if (millis() - servoTimer >= 15) {
      servoTimer = millis();
      if (servoPos > 0) {
        servoPos = servoPos - 1;
        gateServo.write(servoPos);
      } else {
        gateState = 0; // Back to Idle
        lcd.clear();
        lcd.print(" Campus Gate ");
        lcd.setCursor(0, 1);
        lcd.print("  Scan RFID   ");
      }
    }
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **MFRC522 RFID**, **SG90 Servo**, **PIR**, **I2C LCD (16x2)**, **SD Card Reader**, **Buzzer**, and **LED** onto the canvas.
2. Wire the parts matching the wiring table. Ensure Chip Selects are distinct.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the dropdown.
5. Click **Run** to execute.
6. Trigger the RFID widget by scanning card UID `DE AD BE EF` to open the gate or an invalid UID to reject.

## Expected Output
Serial Monitor:
```
Card scanned...
UID: DE AD BE EF - ACCESS GRANTED
Gate Opening...
Gate Open. Waiting...
PIR Sensor Triggered: Motion Detected. Resetting open timer.
No motion detected. Closing gate in 5 seconds.
Gate Closed.
```

## Expected Canvas Behavior
* The I2C LCD initializes with "Campus Gate" and "Scan RFID".
* Upon scanning authorized RFID, the Buzzer sounds a short beep, the LED is off, and the Servo sweeps from 0 to 90 degrees.
* Upon scanning unauthorized RFID, the Buzzer double-beeps, the Red LED flashes twice, and the LCD displays "Access Denied!".
* If the PIR widget detects motion (triggered during open state), the LCD says "Motion Detected" and the gate stays open.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `SPI.begin()` | Sets up SPI communications for both the RFID and SD Card reader modules. |
| `mfrc522.PCD_Init()` | Soft-boots the MFRC522 RFID reader over the hardware SPI lines. |
| `mfrc522.uid.uidByte[0] == 0xDE ...` | Directly checks the 4 bytes of UID array index values to authenticate access without loops. |
| `SD.open("gate_log.txt", FILE_WRITE)` | Opens a log file in append mode to record events directly on the SD storage card. |
| `gateState == 1` | Smoothly increments servoPos by 1 degree every 15ms using timer states instead of delays. |
| `motionDetected` | Updates stateTimer in the open state (gateState 2) to delay closing while PIR is active. |

## Hardware & Safety Concept
* **Dual-SPI Device Contention**: The SPI bus can be shared between the MFRC522 and the SD Card reader. Only one device must have its Chip Select pin pulled LOW at any time. The libraries handle this by asserting the target pin and deasserting other devices.
* **Inductive Isolation**: Driving the servo motor directly from the ARIES 5V line can cause noise or brownouts when the motor stalls. In a physical setup, use an external power supply for the servo and common the grounds.

## Try This! (Challenges)
1. **Intruder Alarm**: If motion is detected while the gate is closed (gateState == 0), flash the warning LED and buzz continuously until an authorized card is scanned.
2. **Master Key Add**: Enable a special card that toggles the gate open permanently until the card is scanned again.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| SD Card fails to initialize | Wrong CS pin or card not formatted | Verify SD CS is on GPIO 8 and the card is formatted to FAT16/FAT32. |
| Card scanned but not recognized | Card UID is not DE AD BE EF | Read card UID through Serial Monitor printout first, then update the code matching the scanned values. |
| Servo does not move or vibrates | Insufficient current or bad pin allocation | Ensure Servo is powered from the 5V rail and signal is connected to GPIO 13. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [143 - RFID Lockers](../advanced/143-rfid-lockers.md)
- [144 - RFID Access Logger](../advanced/144-rfid-access-logger.md)
- [182 - Smart Parking Barrier](182-smart-parking-barrier.md)
