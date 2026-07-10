# 117 - Smart Lock

Create an automated security entryway utilizing an RFID reader (MFRC522) for authentication and a PIR motion sensor to verify entry. Upon scanning a valid card, the door servo rotates to unlock and the PIR sensor tracks if the person passes through.

## Goal
Learn how to chain state transitions across multiple sensor components (RFID scan → unlock state → motion tracking phase → relocking state) without using blocking arrays or loops in interpreted mode.

## What You Will Build
An access control checkpoint:
- **RFID Scan**: User scans their keycard. If the UID matches the master credentials, access is granted.
- **Door Servo**: Swings 90 degrees to unlock the door.
- **PIR Motion Sensor**: Becomes active for 5 seconds after unlocking. If motion is detected, the LCD displays "User Entered" and a green LED pulses.
- **Relock Control**: If no motion is detected within 5 seconds, or immediately after entry is registered, the servo returns to 0 degrees to lock the entry.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| MFRC522 RFID | `mfrc522` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| PIR Sensor | `pir` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| RFID Reader | 3.3V | 3.3V | Power supply |
| RFID Reader | RST | D9 | Reset control |
| RFID Reader | SDA (SS) | D10 | SPI Slave Select |
| RFID Reader | MOSI | D11 | SPI MOSI |
| RFID Reader | MISO | D12 | SPI MISO |
| RFID Reader | SCK | D13 | SPI Clock |
| RFID Reader | GND | GND | Ground reference |
| Servo Motor | Signal | D6 | PWM servo signal |
| PIR Sensor | OUT | D2 | Digital motion output |
| LED | Anode | D5 | Indicator LED |
| Active Buzzer | VCC | D8 | Alarm/Alert feedback |
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

const int RST_PIN    = 9;
const int SS_PIN     = 10;
const int SERVO_PIN  = 6;
const int PIR_PIN    = 2;
const int LED_PIN    = 5;
const int BUZZER_PIN = 8;

MFRC522 rfid(SS_PIN, RST_PIN);
Servo lockServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Master keycard UID bytes (e.g. 0x3F 0x5D 0x8A 0xB9)
const byte AUTH_B0 = 0x3F;
const byte AUTH_B1 = 0x5D;
const byte AUTH_B2 = 0x8A;
const byte AUTH_B3 = 0xB9;

bool isUnlocked = false;
unsigned long unlockTime = 0;
const unsigned long TIMEOUT_MS = 5000;

void setup() {
  Serial.begin(9600);
  SPI.begin();
  rfid.PCD_Init();
  lockServo.attach(SERVO_PIN);
  
  pinMode(PIR_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  lockServo.write(0); // Default Locked
  digitalWrite(LED_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);

  lcd.init();
  lcd.backlight();
  lcd.print("Smart Lock Act.");
  delay(1500);
  lcd.clear();

  Serial.println("Smart Lock Ready. Scan card...");
}

void loop() {
  // If not unlocked, check for new RFID cards
  if (!isUnlocked) {
    lcd.setCursor(0, 0);
    lcd.print("Scan RFID Card  ");
    lcd.setCursor(0, 1);
    lcd.print("Door Locked     ");

    if (rfid.PICC_IsNewCardPresent()) {
      if (rfid.PICC_ReadCardSerial()) {
        byte b0 = rfid.uid.uidByte[0];
        byte b1 = rfid.uid.uidByte[1];
        byte b2 = rfid.uid.uidByte[2];
        byte b3 = rfid.uid.uidByte[3];

        Serial.print("Card UID: ");
        Serial.print(b0, HEX); Serial.print(" ");
        Serial.print(b1, HEX); Serial.print(" ");
        Serial.print(b2, HEX); Serial.print(" ");
        Serial.println(b3, HEX);

        if (b0 == AUTH_B0 && b1 == AUTH_B1 && b2 == AUTH_B2 && b3 == AUTH_B3) {
          Serial.println("ACCESS GRANTED");
          
          isUnlocked = true;
          unlockTime = millis();
          
          lockServo.write(90); // Unlock
          digitalWrite(LED_PIN, HIGH);
          
          // Access beep
          digitalWrite(BUZZER_PIN, HIGH);
          delay(150);
          digitalWrite(BUZZER_PIN, LOW);
          delay(100);
          digitalWrite(BUZZER_PIN, HIGH);
          delay(150);
          digitalWrite(BUZZER_PIN, LOW);

          lcd.clear();
          lcd.setCursor(0, 0);
          lcd.print("Access Granted  ");
          lcd.setCursor(0, 1);
          lcd.print("Wait for Entry  ");
        } else {
          Serial.println("ACCESS DENIED");
          
          // Warning tone
          digitalWrite(BUZZER_PIN, HIGH);
          delay(600);
          digitalWrite(BUZZER_PIN, LOW);

          lcd.clear();
          lcd.setCursor(0, 0);
          lcd.print("Access Denied   ");
          delay(2000);
          lcd.clear();
        }
        
        rfid.PICC_HaltA();
      }
    }
  } else {
    // We are unlocked, monitor PIR and timeout
    int motion = digitalRead(PIR_PIN);
    unsigned long elapsed = millis() - unlockTime;

    if (motion == HIGH) {
      Serial.println("Motion detected! User entered.");
      lcd.setCursor(0, 1);
      lcd.print("User Entered    ");
      delay(1500);

      // Relock immediately
      isUnlocked = false;
      lockServo.write(0);
      digitalWrite(LED_PIN, LOW);
      lcd.clear();
    } else if (elapsed >= TIMEOUT_MS) {
      Serial.println("Entry timeout. Relocking door.");
      
      isUnlocked = false;
      lockServo.write(0); // Relock
      digitalWrite(LED_PIN, LOW);
      
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Timeout Lock    ");
      delay(1500);
      lcd.clear();
    }
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **MFRC522 RFID**, **Servo Motor**, **PIR Sensor**, **LED**, **Active Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Connect RFID: **3.3V** to **3.3V** (or **3V3** pin), **RST** to **D9**, **SDA** (or **SS**) to **D10**, **MOSI** to **D11**, **MISO** to **D12**, **SCK** to **D13**, **GND** to **GND**.
3. Connect Servo: **Signal** to **D6**, **VCC** to **5V**, **GND** to **GND**.
4. Connect PIR: **OUT** to **D2**, **VCC** to **5V**, **GND** to **GND**.
5. Connect LED: **Anode** to **D5** (via resistor if preferred), **Cathode** to **GND**.
6. Connect Buzzer: **VCC** to **D8**, **GND** to **GND**.
7. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
8. Paste code, run, and scan card. Set PIR state to `HIGH` within 5 seconds to test entry, or wait 5 seconds for timeout locking.

## Expected Output

Terminal:
```
Smart Lock Ready. Scan card...
Card UID: 3F 5D 8A B9
ACCESS GRANTED
Motion detected! User entered.
```

LCD Display:
```
Access Granted  
Wait for Entry  
```

## Expected Canvas Behavior
| RFID Card Scan | PIR State (D2) | Elapsed Time (ms) | Servo Position | LED (D5) | LCD Status |
| --- | --- | --- | --- | --- | --- |
| None | LOW | - | 0 deg | LOW | `Scan RFID Card` / `Door Locked` |
| Invalid card | LOW | - | 0 deg | LOW | `Access Denied` |
| Valid card | LOW | < 100 | 90 deg | HIGH | `Access Granted` / `Wait for Entry` |
| Valid card | HIGH | 1500 | 0 deg (after move) | LOW | `User Entered` |
| Valid card | LOW | > 5000 | 0 deg (after timeout)| LOW | `Timeout Lock` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `rfid.uid.uidByte[0..3]` | Extracts individual unique identifier bytes from the scanned physical transponder. |
| `millis() - unlockTime` | Tracks non-blocking elapsed time from the moment authentication succeeded. |
| `lockServo.write(0)` | Moves the physical alignment shaft back to the locking position. |

## Hardware & Safety Concept: Physical Access Control
RFID authentication relies on radio-frequency communication between the reader antenna and the internal chip of the passive transponder card. Combined with a PIR sensor, it forms a double-verification security boundary: ensuring that doors do not hang open indefinitely and verifying that a passage actually took place before locking the mechanism.

## Try This! (Challenges)
1. **Force Entry Alarm**: If motion is detected on D2 *while* the door is locked (`isUnlocked == false`), trigger a continuous warning alarm on D8 and display "INTRUDER ALERT" on the LCD.
2. **Door Contact Sensor**: Add a switch (representing a door sensor) and confirm it closes before allowing the lock servo to return to 0 degrees.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| SPI read fails | Wrong pin mappings for RFID | Confirm D11=MOSI, D12=MISO, D13=SCK, and D10=SS. |
| Servo doesn't turn | Pin D6 not configured or low power | Double-check that `lockServo.attach(6)` is executed in `setup()`. |

## Mode Notes
This multi-stage state logic runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [70 - RFID Servo Unlock](../intermediate/70-rfid-servo-unlock.md)
- [94 - RFID + Servo + LCD](../advanced/94-rfid-servo-lcd.md)
- [109 - Full Access Control](../advanced/109-full-access-control.md)
