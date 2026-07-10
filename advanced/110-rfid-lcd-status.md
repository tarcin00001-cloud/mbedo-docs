# 110 - RFID LCD Status

Scan an RFID card and immediately show "Access Granted" or "Access Denied" on a 16x2 I2C LCD, with a matching green or red LED confirmation light.

## Goal
Learn how to display real-time RFID authentication results on an LCD so that the status is visible to a bystander without needing a connected computer or Serial Monitor.

## What You Will Build
The LCD updates on every card scan:

- **Authorized card**: LCD line 1 shows `Access Granted`, line 2 shows the card UID. Green LED turns on for 2 seconds.
- **Unauthorized card**: LCD line 1 shows `Access Denied`, line 2 shows `Unknown Card`. Red LED turns on for 2 seconds.
- Between scans the LCD shows `Scan card...` on line 1.

**Why D3, D5, D9, D10–D13?** The MFRC522 uses SPI (D10 CS, D11 MOSI, D12 MISO, D13 SCK) plus D9 RST. Pins D3 and D5 drive the indicator LEDs. The LCD shares the I2C bus on A4/A5 with the Arduino.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes (per LED) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| MFRC522 RFID | SDA (CS) | D10 | Chip select (SPI) |
| MFRC522 RFID | SCK | D13 | SPI Clock |
| MFRC522 RFID | MOSI | D11 | SPI Data out |
| MFRC522 RFID | MISO | D12 | SPI Data in |
| MFRC522 RFID | RST | D9 | Reset signal |
| MFRC522 RFID | 3.3V | 3.3V | Power supply |
| MFRC522 RFID | GND | GND | Ground reference |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data bus |
| 16x2 I2C LCD | SCL | A5 | I2C Clock bus |
| 16x2 I2C LCD | GND | GND | Ground reference |
| Green LED | Anode (+) | D3 | Access granted indicator |
| Green LED | Cathode (-) | GND | Via 220 ohm resistor |
| Red LED | Anode (+) | D5 | Access denied indicator |
| Red LED | Cathode (-) | GND | Via 220 ohm resistor |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int RST_PIN   = 9;
const int SS_PIN    = 10;
const int GREEN_PIN = 3;
const int RED_PIN   = 5;

// Authorized card UID — update to match your card
const byte AUTH_B0 = 0xDE;
const byte AUTH_B1 = 0xAD;
const byte AUTH_B2 = 0xBE;
const byte AUTH_B3 = 0xEF;

MFRC522 rfid(SS_PIN, RST_PIN);
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(9600);
  SPI.begin();
  rfid.PCD_Init();

  pinMode(GREEN_PIN, OUTPUT);
  pinMode(RED_PIN,   OUTPUT);
  digitalWrite(GREEN_PIN, LOW);
  digitalWrite(RED_PIN,   LOW);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("RFID LCD Status ");
  lcd.setCursor(0, 1);
  lcd.print("Starting...     ");
  delay(1500);
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Scan card...    ");

  Serial.println("RFID LCD Status Ready. Scan a card...");
}

void loop() {
  if (!rfid.PICC_IsNewCardPresent() || !rfid.PICC_ReadCardSerial()) return;

  byte b0 = rfid.uid.uidByte[0];
  byte b1 = rfid.uid.uidByte[1];
  byte b2 = rfid.uid.uidByte[2];
  byte b3 = rfid.uid.uidByte[3];

  // Build UID string for Serial print
  Serial.print("UID: ");
  Serial.print(b0, HEX); Serial.print(" ");
  Serial.print(b1, HEX); Serial.print(" ");
  Serial.print(b2, HEX); Serial.print(" ");
  Serial.println(b3, HEX);

  if (b0 == AUTH_B0 && b1 == AUTH_B1 && b2 == AUTH_B2 && b3 == AUTH_B3) {

    // Granted
    Serial.println("-> Access Granted");
    digitalWrite(GREEN_PIN, HIGH);

    lcd.setCursor(0, 0);
    lcd.print("Access Granted  ");
    lcd.setCursor(0, 1);
    lcd.print(b0, HEX); lcd.print(" ");
    lcd.print(b1, HEX); lcd.print(" ");
    lcd.print(b2, HEX); lcd.print(" ");
    lcd.print(b3, HEX); lcd.print("    ");

    delay(2000);
    digitalWrite(GREEN_PIN, LOW);

  } else {

    // Denied
    Serial.println("-> Access Denied");
    digitalWrite(RED_PIN, HIGH);

    lcd.setCursor(0, 0);
    lcd.print("Access Denied   ");
    lcd.setCursor(0, 1);
    lcd.print("Unknown Card    ");

    delay(2000);
    digitalWrite(RED_PIN, LOW);
  }

  // Return to idle display
  lcd.setCursor(0, 0);
  lcd.print("Scan card...    ");
  lcd.setCursor(0, 1);
  lcd.print("                ");

  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
  delay(500);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **MFRC522 RFID Reader**, **16x2 I2C LCD**, **Green LED**, and **Red LED** onto the canvas.
2. Wire MFRC522: **SDA** to **D10**, **SCK** to **D13**, **MOSI** to **D11**, **MISO** to **D12**, **RST** to **D9**, **3.3V** to **3.3V**, **GND** to **GND**.
3. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
4. Connect Green LED anode to **D3**, cathode to **GND**.
5. Connect Red LED anode to **D5**, cathode to **GND**.
6. Paste the code into the editor.
7. Use the default Arduino interpreted runtime.
8. Click **Run**.
9. Click the RFID reader on the canvas to simulate card scans and watch the LCD and LEDs respond.

## Expected Output

Terminal (Authorized card):
```
RFID LCD Status Ready. Scan a card...
UID: DE AD BE EF
-> Access Granted
```

Terminal (Unknown card):
```
UID: AA BB CC DD
-> Access Denied
```

LCD Display (Authorized):
```
Access Granted
DE AD BE EF
```

LCD Display (Denied):
```
Access Denied
Unknown Card
```

## Expected Canvas Behavior
| Scanned UID | Match | LCD Line 1 | LCD Line 2 | Green LED | Red LED |
| --- | --- | --- | --- | --- | --- |
| `DE AD BE EF` | Yes | `Access Granted` | UID bytes | ON 2 s | OFF |
| `AA BB CC DD` | No | `Access Denied` | `Unknown Card` | OFF | ON 2 s |
| (idle) | — | `Scan card...` | (blank) | OFF | OFF |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `byte b0 = rfid.uid.uidByte[0]` | Copies each UID byte into a local variable for repeated use without re-indexing the RFID struct. |
| `b0 == AUTH_B0 && b1 == AUTH_B1 && …` | Performs a 4-byte equality check without arrays or loops — fully interpreter-safe. |
| `lcd.print(b0, HEX)` | Prints the byte value in hexadecimal format directly to the LCD (e.g., `DE`). |
| Trailing `"    "` spaces | Ensures LCD lines are fully cleared when shorter strings replace longer ones. |
| `rfid.PICC_HaltA()` | Halts the card so it is not repeatedly re-read on subsequent loop iterations. |

## Hardware & Safety Concept: Visual Status Feedback
Physical access control systems always include a visible indicator (light, LCD, or mechanical indicator) that shows the result of an authentication attempt without requiring a laptop. This makes the system usable by non-technical operators and allows security personnel to spot anomalies from across the room. A green/red LED pair provides immediate intuitive feedback: green means go, red means stop.

## Try This! (Challenges)
1. **Scan Counter**: Add a global `int grantCount = 0` variable. Increment it on each granted scan and display the running total on LCD line 2 during the idle state as `Grants: 3`.
2. **Denied Lockout**: If 3 denied scans occur in sequence, set a `bool locked = true` flag, display `SYSTEM LOCKED` on the LCD, and ignore all scans until the Arduino is reset.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD stays blank | Wrong I2C address | Try changing `0x27` to `0x3F` in the `LiquidCrystal_I2C` constructor. |
| Every card shows "Access Denied" | UID constants do not match your card | Scan your card, read the hex UID from the Terminal, and update the four `AUTH_Bx` constants. |
| Both LEDs flicker | Missing `noTone()` or wrong pins | Confirm GREEN_PIN = D3 and RED_PIN = D5, and verify each `digitalWrite(pin, LOW)` after the delay. |

## Mode Notes
MFRC522 SPI reads, byte comparison, LCD output, and digital output are all supported by MbedO interpreted mode.

## Related Projects
- [68 - RFID Access LED](../intermediate/68-rfid-access-led.md)
- [93 - RFID LED Buzzer](93-rfid-led-buzzer.md)
- [109 - Full Access Control](109-full-access-control.md)
- [111 - Locker System](111-locker-system.md)
