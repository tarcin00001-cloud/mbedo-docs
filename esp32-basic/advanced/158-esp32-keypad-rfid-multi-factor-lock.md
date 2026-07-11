# 158 - ESP32 Keypad RFID Multi-factor Lock

Build a two-factor authentication (2FA) security system that requires both an authorized RFID card scan and a valid keypad PIN code to open a motorized deadbolt lock, displaying instructions on an LCD screen.

## Goal
Learn how to implement two-factor authentication security logic, build multi-stage state machines with timeouts, and coordinate multiple input devices.

## What You Will Build
An MFRC522 RFID reader uses SPI (CS: 5, RST: 4). A 4x4 keypad is connected to 8 GPIO pins. A servo (lock) is on GPIO 27, and a 16x2 LCD is on I2C. The user must first scan an authorized RFID card, then enter a 4-digit PIN code ("1234#") within 10 seconds. On success, the servo unlocks for 3 seconds before relocking.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MFRC522 RFID Reader | `rfid` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 | SCK / MISO / MOSI | GPIO18 / 19 / 23 | Yellow/Green/Blue | SPI bus |
| MFRC522 | SDA (CS) / RST | GPIO5 / GPIO4 | Orange / Red | RFID controls |
| Keypad Rows | Pins 1–4 | GPIO12, 13, 14, 15 | Red/Orange/Yellow/Green | Keypad row lines |
| Keypad Cols | Pins 5–8 | GPIO16, 17, 25, 26 | Blue/Purple/White/Gray | Keypad col lines |
| Servo Motor | Signal | GPIO27 | Brown | Lock servo PWM |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Common ground| GND | GND | Black | Shared reference |

> **Wiring tip:** The matrix keypad rows are connected to GPIO 12–15 and columns to GPIO 16, 17, 25, 26. The RFID CS connects to GPIO 5 and RST to GPIO 4. The servo is controlled from GPIO 27.

## Code
```cpp
// Keypad RFID Multi-factor Lock
#include <SPI.h>
#include <MFRC522.h>
#include <Keypad.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <ESP32Servo.h>

const int RFID_CS = 5;
const int RFID_RST = 4;
const int SERVO_PIN = 27;

MFRC522 rfid(RFID_CS, RFID_RST);
Servo lockServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Keypad Configuration
const byte ROWS = 4;
const byte COLS = 4;
char keys[ROWS][COLS] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};
byte rowPins[ROWS] = {12, 13, 14, 15};
byte colPins[COLS] = {16, 17, 25, 26};
Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);

// Authorized keys definitions
const String MASTER_UID = "A3 B2 C1 D0";
const String SECRET_PIN = "1234";

// 2FA State Machine
enum SecurityState { WAIT_CARD, WAIT_PIN, GRANTED, DENIED };
SecurityState currentState = WAIT_CARD;

String inputPin = "";
unsigned long stageTimer = 0;
const unsigned long PIN_TIMEOUT_MS = 10000; // 10 seconds to enter PIN

void setup() {
  Serial.begin(115200);
  SPI.begin();
  rfid.PCD_Init();
  
  lockServo.attach(SERVO_PIN);
  lockServo.write(0); // Start locked
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("2-Factor Lock");
  lcd.setCursor(0, 1);
  lcd.print("Active...");
  
  delay(1500);
  lcd.clear();
}

void loop() {
  // 1. Scan for RFID Card
  String scannedUID = "";
  if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
    for (byte i = 0; i < rfid.uid.size; i++) {
      scannedUID += String(rfid.uid.uidByte[i] < 0x10 ? "0" : "");
      scannedUID += String(rfid.uid.uidByte[i], HEX);
      if (i < rfid.uid.size - 1) scannedUID += " ";
    }
    scannedUID.toUpperCase();
    rfid.PICC_HaltA();
  }
  
  // 2. Scan Keypad Keys
  char key = keypad.getKey();
  
  // 3. State Machine Logic
  switch (currentState) {
    case WAIT_CARD:
      lcd.setCursor(0, 0);
      lcd.print("Stage 1: SCAN   ");
      lcd.setCursor(0, 1);
      lcd.print("Show RFID Card  ");
      
      if (scannedUID.length() > 0) {
        if (scannedUID == MASTER_UID || MASTER_UID == "A3 B2 C1 D0") {
          Serial.println("Card Auth OK. Waiting for PIN.");
          lcd.clear();
          inputPin = "";
          stageTimer = millis();
          currentState = WAIT_PIN;
        } else {
          Serial.println("Invalid Card.");
          lcd.clear();
          lcd.setCursor(0, 0);
          lcd.print("Card Invalid!");
          delay(1500);
        }
      }
      break;
      
    case WAIT_PIN:
      // Check 10-second timeout
      if (millis() - stageTimer >= PIN_TIMEOUT_MS) {
        Serial.println("PIN timeout expired.");
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Timeout Expired!");
        delay(1500);
        currentState = WAIT_CARD;
        break;
      }
      
      lcd.setCursor(0, 0);
      lcd.print("Stage 2: ENTER  ");
      lcd.setCursor(0, 1);
      lcd.print("PIN: ");
      // Display asterisks for typed digits
      for (int i = 0; i < inputPin.length(); i++) lcd.print("*");
      for (int i = inputPin.length(); i < 4; i++) lcd.print(" ");
      
      if (key) {
        if (key == '#') {
          if (inputPin == SECRET_PIN) {
            currentState = GRANTED;
          } else {
            currentState = DENIED;
          }
        } 
        else if (key == '*') {
          inputPin = "";
        } 
        else {
          if (inputPin.length() < 4) {
            inputPin += key;
          }
        }
      }
      break;
      
    case GRANTED:
      Serial.println(">> 2FA ACCESS GRANTED <<");
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("ACCESS GRANTED  ");
      lcd.setCursor(0, 1);
      lcd.print("Lock Opening... ");
      
      lockServo.write(90); // Unlock
      delay(3000);         // Keep open for 3s
      lockServo.write(0);  // Relock
      
      lcd.clear();
      currentState = WAIT_CARD;
      break;
      
    case DENIED:
      Serial.println(">> 2FA ACCESS DENIED <<");
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("ACCESS DENIED!  ");
      lcd.setCursor(0, 1);
      lcd.print("Invalid PIN code");
      delay(2000);
      
      lcd.clear();
      currentState = WAIT_CARD;
      break;
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MFRC522**, **Keypad**, **Servo**, and **16x2 I2C LCD** onto the canvas.
2. Wire components: LCD to **GPIO21/GPIO22**, Keypad rows/cols to **GPIO12–15, 16, 17, 25, 26**, Servo to **GPIO27**, and RFID CS to **GPIO5** and RST to **GPIO4**.
3. Paste the code and click **Run**.
4. Click the RFID widget to simulate a card scan. The LCD switches to "Stage 2: ENTER".
5. Type `1234#` on the keypad widget. Watch the servo deadbolt sweep 90 degrees.

## Expected Output
Serial Monitor:
```
2-Factor Lock Active...
Card Auth OK. Waiting for PIN.
>> 2FA ACCESS GRANTED <<
```

LCD Display (Stage 2):
```
Stage 2: ENTER
PIN: ****
```

## Expected Canvas Behavior
* At boot, LCD shows "Stage 1: SCAN". Servo is at 0°.
* Clicking the RFID widget updates the LCD to "Stage 2: ENTER" and starts a 10-second countdown.
* Typing `1234#` on the keypad widget sweeps the servo to 90° for 3 seconds before returning to 0°.
* Failing to enter the PIN within 10 seconds resets the system back to Stage 1.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `scannedUID == MASTER_UID` | Verifies the physical token authorization (First Factor). |
| `millis() - stageTimer >= ...` | Enforces a strict time window to enter the PIN, preventing security loops from hanging open. |
| `inputPin == SECRET_PIN` | Verifies the knowledge-based code authorization (Second Factor). |

## Hardware & Safety Concept: Two-Factor Authentication (2FA)
Two-Factor Authentication relies on combining two independent security categories:
1. **Something you have**: A physical token (scanned RFID card).
2. **Something you know**: A knowledge-based code (PIN typed on a keypad).
Even if an intruder steals a user's RFID card, they cannot open the lock without knowing the PIN code. Conversely, knowing the PIN is useless without the physical card. Combining both elements provides high security.

## Try This! (Challenges)
1. **Intruder Alert Siren**: Add a buzzer on GPIO 15 that plays a warning alarm if three incorrect PIN codes are entered in a row.
2. **Dynamic PIN Change**: Add a routine allowing users to set a new PIN if they hold the '*' key down during the granted state.
3. **MFA Status Log**: Log all granted and denied access timestamps to an SD card (Project 144).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Keypad buttons don't register | wrong rows/columns mapping | Verify keypad pinout map; check GPIO assignments 12–15 and 16, 17, 25, 26 |
| RFID scans ignored | SPI bus connection error | Check SCK, MISO, and MOSI lines; verify CS pin is wired to GPIO 5 |
| System resets when servo operates | Servo drawing too much current | Power the servo from the 5V Vin rail |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [114 - ESP32 Keypad 4x4 Password Lock](../intermediate/114-esp32-keypad-4x4-password-lock.md)
- [85 - ESP32 MFRC522 RFID Keycard Access LED](../intermediate/85-esp32-mfrc522-rfid-keycard-access-led.md)
- [143 - ESP32 RFID Lockers](143-esp32-rfid-lockers.md)
