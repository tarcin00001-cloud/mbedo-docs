# 160 - Pico Smart Lock EEPROM

Build a high-security lock system that supports registering a master keycard and stores credentials in non-volatile flash memory (EEPROM) to control a door servo.

## Goal
Learn how to use the `EEPROM` library to store authorized RFID UIDs, control servo motors, and log access status on an LCD display.

## What You Will Build
A keycard-controlled smart lock terminal:
- **MFRC522 RFID Reader**: Scans keycards.
- **EEPROM (Non-volatile Flash)**: Stores the master card UID (4 bytes) at address `30`.
- **Servo Motor (GP10)**: Actuates a physical door latch (0° = Locked, 90° = Unlocked).
- **16x2 I2C LCD (GP4, GP5)**: Displays directions and status logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MFRC522 | SDA / SCK / MOSI / MISO / RST | GP17 / GP18 / GP19 / GP16 / GP22 | SPI Bus |
| Servo Motor | Signal | GP10 | Door latch servo |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>
#include <Servo.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <EEPROM.h>

const int RST_PIN   = 22;
const int SS_PIN    = 17;
const int SERVO_PIN = 10;

MFRC522 mfrc522(SS_PIN, RST_PIN);
Servo lockServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

byte storedUID[4] = {0, 0, 0, 0};
bool masterEnrolled = false;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  SPI.begin();
  mfrc522.PCD_Init();

  lockServo.attach(SERVO_PIN);
  lockServo.write(0); // Lock door initially

  EEPROM.begin(512);

  // Read stored master card UID from EEPROM address 30-33
  storedUID[0] = EEPROM.read(30);
  storedUID[1] = EEPROM.read(31);
  storedUID[2] = EEPROM.read(32);
  storedUID[3] = EEPROM.read(33);

  lcd.init();
  lcd.backlight();

  // Check if a master card has already been enrolled
  if (storedUID[0] != 0xFF && storedUID[0] != 0x00) {
    masterEnrolled = true;
    showLockedScreen();
  } else {
    lcd.setCursor(0, 0);
    lcd.print("No Master Card  ");
    lcd.setCursor(0, 1);
    lcd.print("Scan Card to Reg");
  }
}

void loop() {
  if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
    
    if (!masterEnrolled) {
      // 1. Enroll master card
      storedUID[0] = mfrc522.uid.uidByte[0];
      storedUID[1] = mfrc522.uid.uidByte[1];
      storedUID[2] = mfrc522.uid.uidByte[2];
      storedUID[3] = mfrc522.uid.uidByte[3];

      EEPROM.write(30, storedUID[0]);
      EEPROM.write(31, storedUID[1]);
      EEPROM.write(32, storedUID[2]);
      EEPROM.write(33, storedUID[3]);
      EEPROM.commit();

      masterEnrolled = true;
      
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Master enrolled!");
      lcd.setCursor(0, 1);
      lcd.print("Key Saved       ");
      delay(2000);
      showLockedScreen();
    } else {
      // 2. Access control verification
      bool match = true;
      if (mfrc522.uid.uidByte[0] != storedUID[0]) { match = false; }
      if (mfrc522.uid.uidByte[1] != storedUID[1]) { match = false; }
      if (mfrc522.uid.uidByte[2] != storedUID[2]) { match = false; }
      if (mfrc522.uid.uidByte[3] != storedUID[3]) { match = false; }

      lcd.clear();
      if (match) {
        lcd.setCursor(0, 0);
        lcd.print("Access Approved");
        lcd.setCursor(0, 1);
        lcd.print("Door Unlocked ");
        
        lockServo.write(90); // Unlock door latch
        delay(3000);        // Keep door open for 3 seconds
        lockServo.write(0);  // Lock door latch
      } else {
        lcd.setCursor(0, 0);
        lcd.print("Access Denied!");
        lcd.setCursor(0, 1);
        lcd.print("Unknown keycard");
        delay(2000);
      }
      showLockedScreen();
    }
    mfrc522.PICC_HaltA();
  }
  delay(100);
}

void showLockedScreen() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Secure Lock Node");
  lcd.setCursor(0, 1);
  lcd.print("Scan keycard... ");
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MFRC522 RFID Reader**, **Servo Motor**, and **I2C LCD** onto the canvas.
2. Connect MFRC522 SPI pins, Servo to **GP10**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Scan the first card to save it as the master key, then restart the simulation and verify that the master key stays saved in memory.

## Expected Output

Terminal:
```
Simulation active. Persistent RFID lock controller online.
```

## Expected Canvas Behavior
* Startup (First boot): LCD reads `No Master Card` / `Scan Card to Reg`. Servo is at 0° (Locked).
* Scan Card 1: LCD reads `Master enrolled!`.
* Scan Card 1 again: LCD reads `Access Approved`, Servo rotates to 90° for 3 seconds, then returns to 0°.
* Scan Card 2 (mismatch): LCD reads `Access Denied!`, Servo stays at 0°.
* Restart Simulation: LCD reads `Scan keycard...` directly (skipping the enrollment prompt).

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `lockServo.write(90)` | Rotates the servo motor to 90° to open the physical door lock latch. |

## Hardware & Safety Concept: Door Strike Power Isolation
Dynamic security locks use solenoid electronic door strikes or high-torque servos to lock/unlock doors. Solenoids draw high currents and generate magnetic back-EMF spikes when turned OFF. To protect the Pico, use a optocoupler-isolated relay module to control the strike, and power it from a separate external power block, not the Pico's 3.3V pin.

## Try This! (Challenges)
1. **Reset Master Key**: Connect a button on GP14 that clears the EEPROM when held for 5 seconds to register a new master card.
2. **Breach Warning Alarm**: Sound a buzzer on GP15 if an invalid card is scanned 3 times in a row.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo hums or shakes at the end positions | Angle limit exceeded | Physical servos can bind if driven past their mechanical limits. Adjust the limits in code from `90` to `80` to prevent damage. |

## Mode Notes
This multi-device I2C and SPI project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [87 - Pico RFID Servo](../intermediate/87-pico-rfid-servo.md)
- [124 - Pico Smart Lock](124-pico-smart-lock.md)
- [156 - Pico RFID EEPROM](156-pico-rfid-eeprom.md)
