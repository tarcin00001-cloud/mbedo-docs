# 87 - Pico RFID Servo

Build a smart door lock that unlocks a servo barrier when an authorized RFID card is scanned.

## Goal
Learn how to control servo actuators based on scanned RFID tag validation results.

## What You Will Build
An automatic gate barrier lock:
- **MFRC522 RFID Reader**: Monitors card scans.
- **Servo Motor (GP10)**: Sweeps from 0° (Locked) to 90° (Unlocked) for 3 seconds when the authorized card (`DE 12 A4 C3`) is scanned, then automatically returns to 0°.
- **Green LED (GP14)**: Glows when the door is unlocked.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MFRC522 | SDA (SS) | GP17 | Chip Select |
| MFRC522 | SCK | GP18 | SPI Clock |
| MFRC522 | MOSI | GP19 | Master Out Slave In |
| MFRC522 | MISO | GP16 | Master In Slave Out |
| MFRC522 | RST | GP22 | Reset line |
| Servo Motor | Signal | GP10 | Servo sweep control |
| Servo Motor | Power | VBUS (5V) | Power supply |
| Servo Motor | Ground | GND | Ground return |
| Green LED | Anode | GP14 | Authorized status |
| Green LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>
#include <Servo.h>

const int RST_PIN   = 22;
const int SS_PIN    = 17;
const int SERVO_PIN = 10;
const int GRN_LED   = 14;

MFRC522 mfrc522(SS_PIN, RST_PIN);
Servo myServo;

const byte authUID[4] = {0xDE, 0x12, 0xA4, 0xC3};

void setup() {
  SPI.begin();
  mfrc522.PCD_Init();
  
  myServo.attach(SERVO_PIN);
  myServo.write(0); // Start locked (0 degrees)
  
  pinMode(GRN_LED, OUTPUT);
  digitalWrite(GRN_LED, LOW);
}

void loop() {
  if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
    
    // Check if card matches authUID
    bool match = true;
    if (mfrc522.uid.uidByte[0] != authUID[0]) { match = false; }
    if (mfrc522.uid.uidByte[1] != authUID[1]) { match = false; }
    if (mfrc522.uid.uidByte[2] != authUID[2]) { match = false; }
    if (mfrc522.uid.uidByte[3] != authUID[3]) { match = false; }

    if (match) {
      // Access Granted - Unlock Door
      digitalWrite(GRN_LED, HIGH);
      myServo.write(90); // Turn servo to 90 degrees
      delay(3000);       // Wait 3 seconds
      myServo.write(0);  // Re-lock door
      digitalWrite(GRN_LED, LOW);
    } else {
      // Access Denied - Flashing Red LED can be added
      delay(500);
    }

    mfrc522.PICC_HaltA();
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MFRC522 RFID Reader**, **Servo**, and **Green LED** onto the canvas.
2. Connect MFRC522 SPI pins: **SDA** to **GP17**, **SCK** to **GP18**, **MOSI** to **GP19**, **MISO** to **GP16**, **RST** to **GP22**.
3. Connect Servo to **GP10**, LED to **GP14**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Scan authorized tag on canvas and watch the servo arm rotate to 90° and return.

## Expected Output

Terminal:
```
Simulation active. Door lock controller ready.
```

## Expected Canvas Behavior
* Scan Authorized Card: Green LED (GP14) turns ON, Servo sweeps to 90° (Unlocked). After 3 seconds, Servo returns to 0° (Locked) and LED turns OFF.
* Scan Unauthorized Card: No action.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `myServo.write(90)` | Rotates the servo output shaft to 90 degrees to lift/open the mechanical lock bolt. |
| `myServo.write(0)` | Returns the lock bolt to 0 degrees to re-engage the latch after user passage. |

## Hardware & Safety Concept: Fail-Secure vs. Fail-Safe
In electronic lock design, **Fail-Secure** locks remain locked (bolt extended) when power is cut off, securing the perimeter (standard for main building entries). **Fail-Safe** locks automatically unlock (bolt retracted) when power is cut off, allowing occupant escape during emergency power outages (standard for fire escape doors).

## Try This! (Challenges)
1. **Access Denied Alarm**: Sound a buzzer on GP10 for 500 ms if an unauthorized card is scanned.
2. **Sustained Entry**: Add a button on GP15 that keeps the door unlocked continuously when held down.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo sweeps but struggles to return | Underpowered supply | The current drawn by the servo can overload the Pico. Try powering the servo from an external supply or VBUS (5V USB pin). |

## Mode Notes
This multi-device SPI project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [46 - Pico Pot Servo](../../beginner/46-pico-pot-servo.md)
- [85 - Pico RFID LED](85-pico-rfid-led.md)
