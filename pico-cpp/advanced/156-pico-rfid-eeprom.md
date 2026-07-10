# 156 - Pico RFID EEPROM

Build an RFID access scanner that stores authorized keycard UIDs in the Raspberry Pi Pico's non-volatile flash memory (EEPROM) so permissions persist across power cuts.

## Goal
Learn how to use the `EEPROM` library to read and write bytes in non-volatile flash memory, register new cards dynamically, and display logs on an LCD.

## What You Will Build
An RFID card registration terminal:
- **MFRC522 RFID Reader**: Scans cards over SPI.
- **EEPROM (Non-volatile Flash)**: Stores the 4-byte UID of the registered master card at address `0`.
- **16x2 I2C LCD (GP4, GP5)**: Displays instructions (e.g., "Scan Master Card" or "Card Enrolled").
- **Green LED (GP14) & Red LED (GP15)**: Displays authorization checks.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| LEDs (Green & Red) | `led` | Yes (two LEDs) | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MFRC522 | SDA / SCK / MOSI / MISO / RST | GP17 / GP18 / GP19 / GP16 / GP22 | SPI Bus |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| Green LED | Anode | GP14 | Success indicator |
| Red LED | Anode | GP15 | Failure indicator |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <EEPROM.h>

const int RST_PIN   = 22;
const int SS_PIN    = 17;
const int GREEN_LED = 14;
const int RED_LED   = 15;

MFRC522 mfrc522(SS_PIN, RST_PIN);
LiquidCrystal_I2C lcd(0x27, 16, 2);

byte storedUID[4] = {0, 0, 0, 0};
bool masterEnrolled = false;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  SPI.begin();
  mfrc522.PCD_Init();

  pinMode(GREEN_LED, OUTPUT);
  pinMode(RED_LED, OUTPUT);
  digitalWrite(GREEN_LED, LOW);
  digitalWrite(RED_LED, LOW);

  // Initialize EEPROM size on RP2040
  EEPROM.begin(512);

  lcd.init();
  lcd.backlight();

  // Read stored master card UID from EEPROM address 0-3
  storedUID[0] = EEPROM.read(0);
  storedUID[1] = EEPROM.read(1);
  storedUID[2] = EEPROM.read(2);
  storedUID[3] = EEPROM.read(3);

  // Check if a master card has already been enrolled
  if (storedUID[0] != 0xFF && storedUID[0] != 0x00) {
    masterEnrolled = true;
    lcd.setCursor(0, 0);
    lcd.print("System Active   ");
    lcd.setCursor(0, 1);
    lcd.print("Scan Keycard... ");
  } else {
    lcd.setCursor(0, 0);
    lcd.print("No Master Enrolled");
    lcd.setCursor(0, 1);
    lcd.print("Scan Card to Reg");
  }
}

void loop() {
  if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
    
    if (!masterEnrolled) {
      // 1. Enroll the first scanned card as the master key card
      storedUID[0] = mfrc522.uid.uidByte[0];
      storedUID[1] = mfrc522.uid.uidByte[1];
      storedUID[2] = mfrc522.uid.uidByte[2];
      storedUID[3] = mfrc522.uid.uidByte[3];

      // Save to flash EEPROM
      EEPROM.write(0, storedUID[0]);
      EEPROM.write(1, storedUID[1]);
      EEPROM.write(2, storedUID[2]);
      EEPROM.write(3, storedUID[3]);
      EEPROM.commit(); // Write to physical flash

      masterEnrolled = true;
      
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Enroll Success!");
      lcd.setCursor(0, 1);
      lcd.print("Master Saved    ");
      
      digitalWrite(GREEN_LED, HIGH);
      delay(2000);
      digitalWrite(GREEN_LED, LOW);

      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("System Active   ");
      lcd.setCursor(0, 1);
      lcd.print("Scan Keycard... ");
    } else {
      // 2. Validate scanned cards against master key card
      bool match = true;
      if (mfrc522.uid.uidByte[0] != storedUID[0]) { match = false; }
      if (mfrc522.uid.uidByte[1] != storedUID[1]) { match = false; }
      if (mfrc522.uid.uidByte[2] != storedUID[2]) { match = false; }
      if (mfrc522.uid.uidByte[3] != storedUID[3]) { match = false; }

      lcd.clear();
      if (match) {
        lcd.setCursor(0, 0);
        lcd.print("Access Approved ");
        lcd.setCursor(0, 1);
        lcd.print("Welcome Back!   ");
        digitalWrite(GREEN_LED, HIGH);
        delay(2000);
        digitalWrite(GREEN_LED, LOW);
      } else {
        lcd.setCursor(0, 0);
        lcd.print("Access Denied!  ");
        lcd.setCursor(0, 1);
        lcd.print("Unknown Card    ");
        digitalWrite(RED_LED, HIGH);
        delay(2000);
        digitalWrite(RED_LED, LOW);
      }

      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("System Active   ");
      lcd.setCursor(0, 1);
      lcd.print("Scan Keycard... ");
    }
    mfrc522.PICC_HaltA();
  }
  delay(100);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MFRC522 RFID Reader**, **I2C LCD**, and **two LEDs** onto the canvas.
2. Connect MFRC522 SPI pins, LCD to **GP4/GP5**, and LEDs to **GP14/GP15**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Scan the first card to save it as the master key, then restart the simulation and verify that the master key stays saved in memory.

## Expected Output

Terminal:
```
Simulation active. Non-volatile RFID enrollment engine online.
```

## Expected Canvas Behavior
* Startup (First boot): LCD reads `No Master Enrolled` / `Scan Card to Reg`.
* Scan Card 1: LCD reads `Enroll Success!` / `Master Saved`. Green LED glows.
* Scan Card 1 again: LCD reads `Access Approved`, Green LED glows.
* Scan Card 2 (mismatch): LCD reads `Access Denied!`, Red LED glows.
* Restart Simulation: LCD reads `System Active` directly (skipping the enrollment prompt).

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `EEPROM.commit()` | Saves the modified EEPROM buffer to the RP2040's physical flash memory block. |

## Hardware & Safety Concept: Non-Volatile Data Persistence
Microcontrollers lose all variables stored in RAM when they lose power. Security systems must use non-volatile memory (such as internal Flash emulated as EEPROM, or external I2C EEPROM chips) to store authorized key UIDs so that lock configurations are not wiped out during power outages.

## Try This! (Challenges)
1. **Master Reset Button**: Connect a button on GP16 that wipes the EEPROM values when held for 3 seconds to allow registering a new master key.
2. **Door Release Actuator**: Connect a relay on GP10 and activate it for 3 seconds on approved scans.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Card registration is lost on reboot | `EEPROM.commit()` missing | On RP2040 microcontrollers, you must call `EEPROM.commit()` after writing data to save it to physical flash memory. |

## Mode Notes
This multi-device I2C and SPI project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [85 - Pico RFID LED](../intermediate/85-pico-rfid-led.md)
- [111 - Pico RFID LCD](../intermediate/111-pico-rfid-lcd.md)
- [141 - Pico RFID OLED](141-pico-rfid-oled.md)
