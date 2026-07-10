# 94 - RFID Servo LCD

Build a complete RFID access control gate: an authorized card scan rotates a servo (simulating a deadbolt or barrier) and the LCD displays the card holder status and last scan result.

## Goal
Learn how to combine RFID UID authentication with a servo actuator and a status LCD to create a realistic access control system with persistent state display.

## What You Will Build
The LCD always shows the current door state on line 1 and the last scan result on line 2. When an authorized card is scanned:
- Servo sweeps from 0 to 90 degrees (door opens)
- LCD shows "OPEN" and "GRANTED"
- After 3 seconds the servo returns to 0 degrees (door closes)
- LCD reverts to "LOCKED" and awaits the next scan

When an unauthorized card is scanned, the servo stays locked and the LCD shows "DENIED".

**Why D6, D9, D10–D13?** Pin D6 drives the servo signal. RC522 uses SPI on D9-D13.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| RC522 RFID Reader | `rc522_rfid` | Yes | Yes |
| Servo Motor | `servo_motor` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| RC522 RFID | SDA (CS) | D10 | Chip select (SPI) |
| RC522 RFID | SCK | D13 | SPI Clock |
| RC522 RFID | MOSI | D11 | SPI Data out |
| RC522 RFID | MISO | D12 | SPI Data in |
| RC522 RFID | RST | D9 | Reset signal |
| RC522 RFID | 3.3V | 3.3V | Power supply |
| RC522 RFID | GND | GND | Ground reference |
| Servo Motor | Signal | D6 | PWM control |
| Servo Motor | VCC | 5V | Power supply |
| Servo Motor | GND | GND | Ground reference |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data bus |
| 16x2 I2C LCD | SCL | A5 | I2C Clock bus |
| 16x2 I2C LCD | GND | GND | Ground reference |

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

MFRC522 rfid(SS_PIN, RST_PIN);
Servo gateServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Authorized UIDs
byte authorizedUIDs[][4] = {
  {0xDE, 0xAD, 0xBE, 0xEF},
  {0x01, 0x23, 0x45, 0x67}
};
const int AUTH_COUNT = sizeof(authorizedUIDs) / sizeof(authorizedUIDs[0]);

bool isAuthorized(byte *uid, byte uidSize) {
  if (uidSize != 4) return false;
  for (int i = 0; i < AUTH_COUNT; i++) {
    bool match = true;
    for (int j = 0; j < 4; j++) {
      if (uid[j] != authorizedUIDs[i][j]) { match = false; break; }
    }
    if (match) return true;
  }
  return false;
}

void setLCDStatus(const char* line1, const char* line2) {
  lcd.setCursor(0, 0); lcd.print(line1); lcd.print("                ");
  lcd.setCursor(0, 1); lcd.print(line2); lcd.print("                ");
}

void setup() {
  Serial.begin(9600);
  SPI.begin();
  rfid.PCD_Init();
  
  gateServo.attach(SERVO_PIN);
  gateServo.write(0); // Start locked
  
  lcd.init();
  lcd.backlight();
  setLCDStatus("Door: LOCKED    ", "Scan card...    ");
  
  Serial.println("RFID Servo Gate Ready. Scan a card...");
}

void loop() {
  if (!rfid.PICC_IsNewCardPresent() || !rfid.PICC_ReadCardSerial()) return;
  
  Serial.print("Card UID: ");
  for (byte i = 0; i < rfid.uid.size; i++) {
    Serial.print(rfid.uid.uidByte[i], HEX); Serial.print(" ");
  }
  Serial.println();
  
  if (isAuthorized(rfid.uid.uidByte, rfid.uid.size)) {
    Serial.println("-> ACCESS GRANTED. Opening gate...");
    setLCDStatus("Door: OPEN      ", "GRANTED         ");
    
    // Open gate (0 -> 90 degrees)
    for (int pos = 0; pos <= 90; pos += 5) {
      gateServo.write(pos);
      delay(20);
    }
    
    delay(3000); // Hold open for 3 seconds
    
    Serial.println("-> Closing gate.");
    
    // Close gate (90 -> 0 degrees)
    for (int pos = 90; pos >= 0; pos -= 5) {
      gateServo.write(pos);
      delay(20);
    }
    
    setLCDStatus("Door: LOCKED    ", "Scan card...    ");
    
  } else {
    Serial.println("-> ACCESS DENIED.");
    setLCDStatus("Door: LOCKED    ", "DENIED          ");
    delay(2000);
    setLCDStatus("Door: LOCKED    ", "Scan card...    ");
  }
  
  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
  delay(500);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **RC522 RFID**, **Servo Motor**, and **16x2 I2C LCD** onto the canvas.
2. Wire RC522 per the wiring table.
3. Connect Servo: **Signal** to **D6**, **VCC** to **5V**, **GND** to **GND**.
4. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
5. Paste the code into the editor.
6. Use the default Arduino interpreted runtime.
7. Click **Run**.
8. Click the RFID reader to simulate scans and watch the servo sweep and LCD update.

## Expected Output

Terminal (Authorized):
```
Card UID: DE AD BE EF
-> ACCESS GRANTED. Opening gate...
-> Closing gate.
```

LCD Sequence:
```
Door: LOCKED       ->   Door: OPEN        ->   Door: LOCKED
Scan card...       ->   GRANTED           ->   Scan card...
```

## Expected Canvas Behavior

| Event | Servo Angle | LCD Line 1 | LCD Line 2 |
| --- | --- | --- | --- |
| Startup / Idle | 0 deg (Locked) | `Door: LOCKED` | `Scan card...` |
| Authorized scan | 0 -> 90 deg sweep | `Door: OPEN` | `GRANTED` |
| 3s hold open | 90 deg | `Door: OPEN` | `GRANTED` |
| Auto-close | 90 -> 0 deg sweep | `Door: LOCKED` | `Scan card...` |
| Unauthorized scan | 0 deg (no change) | `Door: LOCKED` | `DENIED` |

## Code Walkthrough

| Expression | What It Does |
| --- | --- |
| `for (int pos = 0; pos <= 90; pos += 5)` | Sweeps the servo in 5-degree steps with 20 ms delays for a smooth, visible motion. |
| `setLCDStatus()` helper | Centralizes LCD writes so both lines are updated in one call, with trailing spaces to clear stale text. |

## Hardware & Safety Concept: Servo Return-to-Home
Always design motorized gate systems with a **fail-safe closed** default. The servo is initialized at 0 degrees in `setup()`. If the Arduino resets or loses power, the gate should return to the closed/locked position, not remain open. This is why the servo is explicitly written to `0` in `setup()` before any card is scanned.

## Try This! (Challenges)
1. **Access Counter**: Track how many successful unlocks have occurred. Display the count on LCD line 1 as `Unlocks: 5`.
2. **Lockout Mode**: If 3 consecutive unauthorized scans are detected, activate a lockout for 30 seconds during which no card is accepted, and blink an LED rapidly.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo jitters at 0 degrees | Signal noise on D6 | Add a small capacitor (100 uF) across the servo VCC and GND lines to stabilize power. |
| LCD shows stale characters | Missing trailing spaces | Ensure the `setLCDStatus()` helper appends enough spaces to clear any longer previous strings. |

## Mode Notes
These patterns (RFID UID auth, servo sweep, and LCD state display) are supported by MbedO interpreted mode.

## Related Projects
- [70 - RFID Servo Unlock](../intermediate/70-rfid-servo-unlock.md)
- [93 - RFID LED Buzzer](93-rfid-led-buzzer.md)
- [95 - BT Alarm ArmDisarm](95-bt-alarm-armdisarm.md)
