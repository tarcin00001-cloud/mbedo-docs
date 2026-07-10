# 70 - RFID Servo Unlock

Rotate a servo motor lock mechanism automatically when an authorized RFID card is scanned.

## Goal
Learn how to couple a secure RFID identification system with a servo motor actuator to model an electronic deadbolt or access barrier.

## What You Will Build
When a card is swiped:
- If the card UID matches the master key (`DE AD BE EF`), the servo motor connected to D5 rotates to 90 degrees (unlocked position) and holds for 3 seconds before automatically returning to 0 degrees (locked position).
- If the card does not match, the servo remains locked at 0 degrees.

**Why D5?** Pin D5 is a PWM channel used to drive the servo motor position.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| MFRC522 Module | SDA (SS) | D10 | SPI Slave Select |
| MFRC522 Module | SCK | D13 | SPI Clock |
| MFRC522 Module | MOSI | D11 | SPI MOSI |
| MFRC522 Module | MISO | D12 | SPI MISO |
| MFRC522 Module | RST | D9 | Reset |
| MFRC522 Module | VCC | 5V | Power supply (Caution: 3.3V on hardware) |
| MFRC522 Module | GND | GND | Ground |
| Servo Motor | VCC | 5V | Power (red wire) |
| Servo Motor | SIG | D5 | Control pin (orange wire) |
| Servo Motor | GND | GND | Ground (brown wire) |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>
#include <Servo.h>

#define SS_PIN 10
#define RST_PIN 9
#define SERVO_PIN 5

MFRC522 rfid(SS_PIN, RST_PIN);
Servo lockServo;

// Authorized Master Card UID
const byte MASTER_UID[] = {0xDE, 0xAD, 0xBE, 0xEF};

void setup() {
  Serial.begin(9600);
  SPI.begin();
  rfid.PCD_Init();
  
  lockServo.attach(SERVO_PIN);
  lockServo.write(0); // Start in Locked position (0 degrees)
  
  Serial.println("Smart Lock Ready. Scan card to open...");
}

void loop() {
  // Check for new cards
  if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
    
    bool isAuthorized = true;
    for (byte i = 0; i < 4; i++) {
      if (rfid.uid.uidByte[i] != MASTER_UID[i]) {
        isAuthorized = false;
        break;
      }
    }
    
    if (isAuthorized) {
      Serial.println("[SUCCESS] Unlocked! Access Granted.");
      
      // Rotate servo to unlocked position (90 degrees)
      lockServo.write(90);
      
      // Keep open for 3 seconds
      delay(3000);
      
      // Relock the door
      Serial.println("Locking door...");
      lockServo.write(0);
    } else {
      Serial.println("[DENIED] Key card invalid!");
      // Double check servo remains locked
      lockServo.write(0);
    }
    
    rfid.PICC_HaltA();
    rfid.PCD_StopCrypto1();
  }
  
  delay(100);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **RFID Reader (MFRC522)**, and **Servo Motor** onto the canvas.
2. Connect MFRC522: **SDA** to **D10**, **SCK** to **D13**, **MOSI** to **D11**, **MISO** to **D12**, **RST** to **D9**, **VCC** to **5V**, and **GND** to **GND**.
3. Connect Servo **VCC** to **5V**, **SIG** to **D5**, and **GND** to **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Click the RFID Reader component to trigger a simulated card swipe.
8. Watch the Servo rotate to 90 degrees, pause, and then return to 0 degrees.

## Expected Output

Terminal:
```
Smart Lock Ready. Scan card to open...
[SUCCESS] Unlocked! Access Granted.
Locking door...
...
```

### Expected Canvas Behavior

| Swiped Card UID | Match Check | Servo Position (0-180) | Physical Gate state |
| --- | --- | --- | --- |
| DE AD BE EF | Match | Rotates D5 to 90 for 3s -> 0 | Door Opens then Relocks |
| Other | Mismatch | Remains D5 at 0 | Door stays Locked |

The servo sweeps from 0 to 90 degrees instantly on the canvas when the approved card is swiped.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `lockServo.attach(SERVO_PIN)` | Attaches the servo driver library to pin D5. |
| `lockServo.write(90)` | Rotates the servo shaft to the 90-degree unlocked coordinate. |
| `delay(3000)` | Keeps the servo in the unlocked state for 3 seconds before executing subsequent statements to relock. |

## Hardware & Safety Concept: Current Draws & External Supplies
Servo motors and actuators pull significant current surge spikes when rotating under physical loads.
- Trying to power a large servo motor directly from the Arduino's 5V pin can cause the Arduino to reset or freeze due to voltage drops (brownouts).
- For physical hardware reliability, always power the servo from an **external 5V to 6V power supply**, connecting the external supply ground (-) to both the servo ground and the Arduino GND pin (common ground).

## Try This! (Challenges)
1. **Extend Open Time**: Change the code to keep the lock open for `5` seconds.
2. **Status LEDs**: Add a Red LED (D7) and Green LED (D8) to visually display the lock status: Red ON when locked, Green ON when unlocked.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo does not turn when swipe succeeds | Control wire loose | Verify the servo SIG wire is connected to D5. |
| Arduino board resets on swipe | High current draw from motor | In physical builds, disconnect the servo VCC from the Arduino 5V and power the motor from an external 5V power supply. |

## Mode Notes
These patterns (RFID logic driving servo library position modifications) are supported by MbedO interpreted mode.

## Related Projects
- [46 - Potentiometer Servo](46-potentiometer-servo.md)
- [68 - RFID Access LED](68-rfid-access-led.md)
- [71 - RFID Relay Door](71-rfid-relay-door.md)
