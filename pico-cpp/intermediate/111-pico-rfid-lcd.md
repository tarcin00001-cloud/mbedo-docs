# 111 - Pico RFID LCD

Build an RFID access scanner that displays user names and entry logs on a 16x2 I2C LCD screen.

## Goal
Learn how to parse RFID card UIDs, match them to user database indices, and show custom greetings on I2C displays.

## What You Will Build
An office check-in display console:
- **MFRC522 RFID Reader**: Reads card/fob UIDs.
- **16x2 I2C LCD (GP4, GP5)**: Displays visitor greeting messages (e.g. "Welcome, Alice") on authorized matches, and "Access Denied" on mismatches.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MFRC522 | SDA (SS) | GP17 | Chip Select |
| MFRC522 | SCK | GP18 | SPI Clock |
| MFRC522 | MOSI | GP19 | Master Out Slave In |
| MFRC522 | MISO | GP16 | Master In Slave Out |
| MFRC522 | RST | GP22 | Reset line |
| I2C LCD | VCC | 5V | LCD Power |
| I2C LCD | SDA | GP4 | I2C Data |
| I2C LCD | SCL | GP5 | I2C Clock |
| I2C LCD | GND | GND | Ground return |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int RST_PIN = 22;
const int SS_PIN  = 17;

MFRC522 mfrc522(SS_PIN, RST_PIN);
LiquidCrystal_I2C lcd(0x27, 16, 2);

// User UIDs
const byte aliceUID[4] = {0xDE, 0x12, 0xA4, 0xC3};
const byte bobUID[4]   = {0xF5, 0x88, 0xBB, 0xAA};

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  SPI.begin();
  mfrc522.PCD_Init();
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Scan Keycard...");
}

void loop() {
  if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
    lcd.clear();
    
    // Check Alice
    bool isAlice = true;
    if (mfrc522.uid.uidByte[0] != aliceUID[0]) { isAlice = false; }
    if (mfrc522.uid.uidByte[1] != aliceUID[1]) { isAlice = false; }
    if (mfrc522.uid.uidByte[2] != aliceUID[2]) { isAlice = false; }
    if (mfrc522.uid.uidByte[3] != aliceUID[3]) { isAlice = false; }

    // Check Bob
    bool isBob = true;
    if (mfrc522.uid.uidByte[0] != bobUID[0]) { isBob = false; }
    if (mfrc522.uid.uidByte[1] != bobUID[1]) { isBob = false; }
    if (mfrc522.uid.uidByte[2] != bobUID[2]) { isBob = false; }
    if (mfrc522.uid.uidByte[3] != bobUID[3]) { isBob = false; }

    lcd.setCursor(0, 0);
    if (isAlice) {
      lcd.print("Welcome, Alice");
      lcd.setCursor(0, 1);
      lcd.print("Access Granted  ");
    } 
    else if (isBob) {
      lcd.print("Welcome, Bob");
      lcd.setCursor(0, 1);
      lcd.print("Access Granted  ");
    } 
    else {
      lcd.print("Access Denied!");
      lcd.setCursor(0, 1);
      lcd.print("Unknown User    ");
    }

    delay(2500); // Display greeting message for 2.5 seconds
    
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Scan Keycard...");

    mfrc522.PICC_HaltA();
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MFRC522 RFID Reader**, and **I2C LCD** onto the canvas.
2. Connect MFRC522 SPI pins: **SDA** to **GP17**, **SCK** to **GP18**, **MOSI** to **GP19**, **MISO** to **GP16**, **RST** to **GP22**.
3. Connect LCD to **GP4/GP5**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Scan different card tags on canvas and watch the personalized LCD greetings.

## Expected Output

Terminal:
```
Simulation active. LCD access logging system active.
```

## Expected Canvas Behavior
* Scan Card 1 (`DE 12 A4 C3`): LCD reads `Welcome, Alice` / `Access Granted`.
* Scan Card 2 (`F5 88 BB AA`): LCD reads `Welcome, Bob` / `Access Granted`.
* Scan Unknown Card: LCD reads `Access Denied!` / `Unknown User`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `lcd.print("Welcome, Alice")` | Displays a personalized greeting message on the screen when a matching card UID is scanned. |

## Hardware & Safety Concept: Database Memory Limits
In school and office settings, access databases contain thousands of authorized card UIDs. Storing large arrays of UID strings directly inside the microcontroller's RAM (like the Pico's 264KB SRAM) can quickly exhaust memory. Commercial systems store user UIDs in external Flash memory chips, or send scanned UIDs over Ethernet/Wi-Fi to a server database for remote validation.

## Try This! (Challenges)
1. **Access Tone Indicator**: Connect a buzzer on GP14 and beep twice on success, and play one long low buzz on failure.
2. **Access count tracker**: Add a counter that keeps track of the total number of entries and displays it on the screen.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Display shows "Scan Keycard..." but doesn't react to tags | SPI bus speed too high | Ensure SPI lines are wired correctly. The MFRC522 library automatically manages SPI speed configs during boot. |

## Mode Notes
This multi-device I2C and SPI project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [84 - Pico RFID Serial](../intermediate/84-pico-rfid-serial.md)
- [85 - Pico RFID LED](../intermediate/85-pico-rfid-led.md)
- [101 - Pico Automatic Gate](101-pico-automatic-gate.md)
