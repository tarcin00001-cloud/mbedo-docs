# 111 - Locker System

Scan an RFID card to unlock a simulated locker: a matching UID sweeps a servo to the open position, displays a confirmation on the LCD, and plays a brief confirmation tone. A wrong card shows an error and stays locked.

## Goal
Learn how to integrate RFID authentication, servo actuation, LCD status display, and audible feedback into a single coherent locker/safe system using only interpreted-mode-safe patterns.

## What You Will Build
On every card scan the system responds in two ways:

- **Authorized card**: LCD shows `UNLOCKED` on line 1 and the UID on line 2. Servo moves to 90 degrees (open). Buzzer plays a short confirmation beep. After 3 seconds the servo returns to 0 degrees (locked) and the LCD reverts to `LOCKED — Scan card`.
- **Unauthorized card**: LCD shows `DENIED` on line 1 and `Wrong card` on line 2. Servo stays locked. Buzzer plays a low-pitched warning tone. LCD reverts to idle after 2 seconds.

**Why D6, D8, D9, D10–D13?** The MFRC522 uses SPI (D10 CS, D11 MOSI, D12 MISO, D13 SCK) plus D9 RST. Pin D6 is the servo PWM signal. Pin D8 drives the buzzer. The LCD shares I2C on A4/A5.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes |

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
| Servo Motor | Signal | D6 | PWM control |
| Servo Motor | VCC | 5V | Power supply |
| Servo Motor | GND | GND | Ground reference |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data bus |
| 16x2 I2C LCD | SCL | A5 | I2C Clock bus |
| 16x2 I2C LCD | GND | GND | Ground reference |
| Buzzer | + | D8 | Signal pin |
| Buzzer | - | GND | Ground reference |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>
#include <Servo.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int RST_PIN   = 9;
const int SS_PIN    = 10;
const int SERVO_PIN = 6;
const int BUZ_PIN   = 8;

// Authorized locker UID — update to match your card
const byte AUTH_B0 = 0xDE;
const byte AUTH_B1 = 0xAD;
const byte AUTH_B2 = 0xBE;
const byte AUTH_B3 = 0xEF;

MFRC522 rfid(SS_PIN, RST_PIN);
Servo lockerServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(9600);
  SPI.begin();
  rfid.PCD_Init();

  lockerServo.attach(SERVO_PIN);
  lockerServo.write(0); // Start locked

  pinMode(BUZ_PIN, OUTPUT);
  noTone(BUZ_PIN);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Locker System   ");
  lcd.setCursor(0, 1);
  lcd.print("Starting...     ");
  delay(1500);

  lcd.setCursor(0, 0);
  lcd.print("LOCKED          ");
  lcd.setCursor(0, 1);
  lcd.print("Scan card...    ");

  Serial.println("Locker System Ready. Scan your card...");
}

void loop() {
  if (!rfid.PICC_IsNewCardPresent() || !rfid.PICC_ReadCardSerial()) return;

  byte b0 = rfid.uid.uidByte[0];
  byte b1 = rfid.uid.uidByte[1];
  byte b2 = rfid.uid.uidByte[2];
  byte b3 = rfid.uid.uidByte[3];

  Serial.print("UID: ");
  Serial.print(b0, HEX); Serial.print(" ");
  Serial.print(b1, HEX); Serial.print(" ");
  Serial.print(b2, HEX); Serial.print(" ");
  Serial.println(b3, HEX);

  if (b0 == AUTH_B0 && b1 == AUTH_B1 && b2 == AUTH_B2 && b3 == AUTH_B3) {

    // UNLOCK
    Serial.println("-> UNLOCKED");

    lcd.setCursor(0, 0);
    lcd.print("UNLOCKED        ");
    lcd.setCursor(0, 1);
    lcd.print(b0, HEX); lcd.print(" ");
    lcd.print(b1, HEX); lcd.print(" ");
    lcd.print(b2, HEX); lcd.print(" ");
    lcd.print(b3, HEX); lcd.print("    ");

    lockerServo.write(90); // Open latch

    tone(BUZ_PIN, 1200, 120);
    delay(200);
    noTone(BUZ_PIN);

    delay(3000); // Locker stays open 3 seconds

    lockerServo.write(0); // Re-lock
    Serial.println("-> Re-locked.");

    lcd.setCursor(0, 0);
    lcd.print("LOCKED          ");
    lcd.setCursor(0, 1);
    lcd.print("Scan card...    ");

  } else {

    // DENIED
    Serial.println("-> DENIED");

    lcd.setCursor(0, 0);
    lcd.print("DENIED          ");
    lcd.setCursor(0, 1);
    lcd.print("Wrong card      ");

    tone(BUZ_PIN, 400, 500);
    delay(600);
    noTone(BUZ_PIN);

    delay(1500);

    lcd.setCursor(0, 0);
    lcd.print("LOCKED          ");
    lcd.setCursor(0, 1);
    lcd.print("Scan card...    ");
  }

  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
  delay(500);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **MFRC522 RFID Reader**, **Servo Motor**, **16x2 I2C LCD**, and **Buzzer** onto the canvas.
2. Wire MFRC522: **SDA** → **D10**, **SCK** → **D13**, **MOSI** → **D11**, **MISO** → **D12**, **RST** → **D9**, **3.3V** → **3.3V**, **GND** → **GND**.
3. Connect Servo: **Signal** → **D6**, **VCC** → **5V**, **GND** → **GND**.
4. Connect LCD: **VCC** → **5V**, **SDA** → **A4**, **SCL** → **A5**, **GND** → **GND**.
5. Connect Buzzer: **+** → **D8**, **-** → **GND**.
6. Paste the code into the editor.
7. Use the default Arduino interpreted runtime.
8. Click **Run**.
9. Click the RFID reader on the canvas to simulate scans and watch the servo, LCD, and buzzer respond.

## Expected Output

Terminal (Authorized card):
```
Locker System Ready. Scan your card...
UID: DE AD BE EF
-> UNLOCKED
-> Re-locked.
```

Terminal (Wrong card):
```
UID: AA BB CC DD
-> DENIED
```

LCD Display (Unlocked):
```
UNLOCKED
DE AD BE EF
```

LCD Display (Denied):
```
DENIED
Wrong card
```

## Expected Canvas Behavior
| Event | Servo | LCD Line 1 | LCD Line 2 | Buzzer |
| --- | --- | --- | --- | --- |
| Startup / Idle | 0 deg (Locked) | `LOCKED` | `Scan card...` | Silent |
| Authorized scan | 0 → 90 deg | `UNLOCKED` | UID bytes | Short 1200 Hz beep |
| 3 s open hold | 90 deg | `UNLOCKED` | UID bytes | Silent |
| Auto re-lock | 90 → 0 deg | `LOCKED` | `Scan card...` | Silent |
| Denied scan | 0 deg | `DENIED` | `Wrong card` | 500 ms 400 Hz tone |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `lockerServo.write(0)` in `setup()` | Initializes the servo to the closed/locked position before any card is scanned (fail-safe default). |
| `b0 == AUTH_B0 && b1 == AUTH_B1 && …` | Four-byte UID comparison without arrays or loops — safe for the MbedO interpreter. |
| `tone(BUZ_PIN, 1200, 120)` | Plays a 1200 Hz tone for 120 ms to give a short, pleasant unlock confirmation chime. |
| `delay(3000)` | Simulates the time a user has to open the locker before the latch automatically re-engages. |
| `rfid.PICC_HaltA()` | Halts the detected card so it is not continuously re-read on the next loop iteration. |

## Hardware & Safety Concept: Auto-Relocking
Lockers and safes should always re-lock automatically after a timeout, even if the user forgets to close them. The `delay(3000)` and the subsequent `lockerServo.write(0)` implement this behavior in firmware. In a production system this timeout would be driven by a non-blocking timer (using `millis()`) so the microcontroller can still respond to other inputs while counting down.

## Try This! (Challenges)
1. **Open Duration Countdown**: Print a countdown (`3…`, `2…`, `1…`) on LCD line 2 during the unlock period, using three sequential `delay(1000)` calls with `lcd.print()`.
2. **Audit Log**: Track how many times the locker has been opened with a global `int openCount = 0` variable. Display `Opened: N` on the idle LCD line 2 so the count is always visible.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo does not move | No `attach()` call | Confirm `lockerServo.attach(SERVO_PIN)` is in `setup()`. |
| LCD shows garbage | I2C bus conflict | Ensure both LCD SDA and RFID are not sharing a non-I2C pin. |
| Authorized card shows DENIED | UID bytes mismatch | Scan your card, read the Terminal UID output, and update the four `AUTH_Bx` constants. |

## Mode Notes
SPI RFID reads, servo writes, I2C LCD output, and `tone()` are all supported by MbedO interpreted mode.

## Related Projects
- [70 - RFID Servo Unlock](../intermediate/70-rfid-servo-unlock.md)
- [94 - RFID Servo LCD](94-rfid-servo-lcd.md)
- [109 - Full Access Control](109-full-access-control.md)
- [110 - RFID LCD Status](110-rfid-lcd-status.md)
