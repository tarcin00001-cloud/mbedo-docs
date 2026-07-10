# 87 - ESP32 MFRC522 RFID Servo Door Unlock

Build an automated smart lock system that unlocks a physical deadbolt using a servo motor when an authorized RFID tag is scanned.

## Goal
Learn how to actuate physical locks using servo motors in response to authorized digital security events.

## What You Will Build
An MFRC522 RFID reader connected via SPI. A servo motor on GPIO 13 acts as a door lock latch. When an authorized card is scanned, the servo sweeps from 0° (locked) to 90° (unlocked), pauses for 3 seconds to let the user open the door, and then sweeps back to 0° to lock the door.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MFRC522 RFID Reader Module | `rfid_reader` | Yes | Yes |
| RFID Key Fob / Card (13.56MHz) | `rfid_tag` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 Module | 3.3V | 3V3 | Red | Power |
| MFRC522 Module | GND | GND | Black | Ground |
| MFRC522 Module | RST (Reset) | GPIO22 | Purple | Reset line |
| MFRC522 Module | SDA (SS) | GPIO5 | Orange | Chip Select |
| MFRC522 Module | SCK | GPIO18 | Yellow | SPI Clock |
| MFRC522 Module | MISO | GPIO19 | Green | SPI MISO |
| MFRC522 Module | MOSI | GPIO23 | Blue | SPI MOSI |
| Servo Motor | Signal | GPIO13 | Orange | PWM control signal |
| Servo Motor | VCC | 5V (Vin) | Red | Motor power |
| Servo Motor | GND | GND | Black | Ground reference |

> **Wiring tip:** Connect the servo signal to **GPIO 13** so it does not conflict with the SPI interface pins (5, 18, 19, 23). Power the servo from the ESP32 5V (Vin) pin.

## Code
```cpp
// MFRC522 RFID Servo Door Unlock
#include <SPI.h>
#include <MFRC522.h>
#include <ESP32Servo.h>

#define SS_PIN 5
#define RST_PIN 22
const int SERVO_PIN = 13;

MFRC522 rfid(SS_PIN, RST_PIN);
Servo lockServo;

// Authorised Master Key UID
const byte MASTER_UID[] = {0x83, 0xA2, 0xC3, 0xE4};

void setup() {
  Serial.begin(115200);
  SPI.begin();
  rfid.PCD_Init();
  
  lockServo.attach(SERVO_PIN);
  lockServo.write(0); // Start in LOCKED position (0 degrees)
  
  Serial.println("RFID Door Lock Active. Scan Card...");
}

void loop() {
  if ( ! rfid.PICC_IsNewCardPresent()) return;
  if ( ! rfid.PICC_ReadCardSerial()) return;
  
  bool accessGranted = true;
  
  for (byte i = 0; i < rfid.uid.size; i++) {
    if (rfid.uid.uidByte[i] != MASTER_UID[i]) {
      accessGranted = false;
    }
  }
  
  if (accessGranted) {
    Serial.println("Access Granted! Unlocking door...");
    
    // Rotate servo to unlock (90 degrees)
    lockServo.write(90);
    delay(3000); // Keep door unlocked for 3 seconds
    
    Serial.println("Locking door...");
    // Rotate servo back to lock (0 degrees)
    lockServo.write(0);
  } else {
    Serial.println("Access Denied! Door remains locked.");
    // Double pulse servo to wiggle the lock handle slightly as feedback
    lockServo.write(10);
    delay(150);
    lockServo.write(0);
    delay(150);
    lockServo.write(10);
    delay(150);
    lockServo.write(0);
  }
  
  delay(1000);
  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MFRC522 Reader**, and **Servo** onto the canvas.
2. Connect MFRC522 SPI lines. Connect the Servo signal pin to **GPIO13**.
3. Paste the code and click **Run**.
4. Drag a card tag to the reader widget. If it matches the Master UID, watch the servo turn 90 degrees and then return back to 0.

## Expected Output
Serial Monitor:
```
RFID Door Lock Active. Scan Card...
Access Granted! Unlocking door...
Locking door...
Access Denied! Door remains locked.
```

## Expected Canvas Behavior
* Scanning the matching card tag wiggles the servo to 90° for 3 seconds, then returns it to 0°.
* Scanning a non-matching card tag wiggles the servo slightly (0° to 10° and back twice) to simulate a locked handle rattle, then returns it to 0°.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `lockServo.write(90)` | Rotates the servo motor shaft to 90°, representing the unlocked state. |
| `delay(3000)` | Holds the servo position for 3 seconds to allow transit time through the door. |
| `lockServo.write(10)` | Creates a physical feedback vibration wiggle when a card is rejected. |

## Hardware & Safety Concept: Electromechanical Actuators and Lock Fail-safes
Physical door locks must include manual overrides (e.g. key overrides or physical thumb-turns) so that if the power fails or the microcontroller halts, users can still escape the room. Servo-based locks usually pull a pin or slide a latch rather than directly blocking the handle mechanism, allowing manual safety release.

## Try This! (Challenges)
1. **Unlocking LED Indicator**: Add Green and Red LEDs (Project 85) to indicate lock status alongside the servo.
2. **Door Switch sensor**: Connect a limit switch to GPIO 4. Only re-lock the door once the switch registers that the door has opened and closed.
3. **Auto Lock Timer dial**: Connect a potentiometer (GPIO 34) that allows adjusting the unlock time window from 1 to 10 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo twitches but door doesn't open | Mechanical binding or low current | Verify servo power is connected to 5V (Vin) rail; check for physical obstruction |
| Servo does not lock back | Code stuck in infinite loop | Verify OneWire or SPI loops are not halting setup |
| Inverted lock state | Motor is oriented backwards | Change code values (e.g., 0 for unlock, 90 for lock) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [85 - ESP32 MFRC522 RFID Keycard Access LED](85-esp32-mfrc522-rfid-keycard-access-led.md)
- [86 - ESP32 MFRC522 RFID Keycard Access Buzzer](86-esp32-mfrc522-rfid-keycard-access-buzzer.md)
- [51 - ESP32 Servo Motor 180° Sweep](51-esp32-servo-motor-180-sweep.md)
