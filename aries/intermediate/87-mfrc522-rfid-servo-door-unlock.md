# 87 - MFRC522 RFID Servo Door Unlock

Build an automated smart lock system that verifies RFID keycards, rotates a servo motor to 90 degrees to unlock a door, and automatically re-locks it after 5 seconds.

## Goal
Learn how to coordinate RFID readers and servo actuators using SPI and PWM communication, building an automated lock sequence without using loops inside C++ code.

## What You Will Build
An MFRC522 RFID reader is connected via SPI. A Servo Motor is connected to GPIO 13 of the ARIES v3 board. When an authorized card is scanned, the servo rotates to 90° (unlocked state), waits 5 seconds, and automatically rotates back to 0° (locked state).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MFRC522 RFID Reader Module | `rfid_reader` | Yes | Yes |
| RFID Key Fob / Card (13.56MHz) | `rfid_tag` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes (on GPIO 13) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 Module | 3.3V | 3V3 | Red | Power |
| MFRC522 Module | GND | GND | Black | Ground |
| MFRC522 Module | RST (Reset) | GPIO 9 | Purple | Reset line |
| MFRC522 Module | SDA (SS) | GPIO 10 | Orange | Chip Select |
| MFRC522 Module | SCK | SPI SCK | Yellow | SPI Clock |
| MFRC522 Module | MISO | SPI MISO | Green | SPI MISO |
| MFRC522 Module | MOSI | SPI MOSI | Blue | SPI MOSI |
| Servo Motor | Signal | GPIO 13 | Yellow | Servo PWM control line |
| Servo Motor | VCC | 5V | Red | Servo power supply |
| Servo Motor | GND | GND | Black | Ground reference |

> **Wiring tip:** Servo motors require 5V on VCC to provide full physical torque. Ensure the servo is connected to the 5V rail on the ARIES board, and that grounds are shared.

## Code
```cpp
// MFRC522 RFID Servo Door Unlock
#include <SPI.h>
#include <MFRC522.h>
#include <Servo.h>

#define SS_PIN 10
#define RST_PIN 9
const int SERVO_PIN = 13;

MFRC522 rfid(SS_PIN, RST_PIN);
Servo myServo;

// Authorized Master Key UID bytes (Change this to match your tag)
const byte MASTER_0 = 0x83;
const byte MASTER_1 = 0xA2;
const byte MASTER_2 = 0xC3;
const byte MASTER_3 = 0xE4;

void setup() {
  Serial.begin(115200);
  SPI.begin();
  rfid.PCD_Init();
  
  myServo.attach(SERVO_PIN);
  myServo.write(0); // Start at locked position (0 degrees)
  
  Serial.println("RFID Servo Door Lock active. Scan Card...");
}

void loop() {
  if ( ! rfid.PICC_IsNewCardPresent()) return;
  if ( ! rfid.PICC_ReadCardSerial()) return;
  
  Serial.print("Scanned UID: ");
  Serial.print(rfid.uid.uidByte[0], HEX);
  Serial.print(" ");
  Serial.print(rfid.uid.uidByte[1], HEX);
  Serial.print(" ");
  Serial.print(rfid.uid.uidByte[2], HEX);
  Serial.print(" ");
  Serial.println(rfid.uid.uidByte[3], HEX);
  
  // Compare scanned UID bytes with master key
  if (rfid.uid.uidByte[0] == MASTER_0 &&
      rfid.uid.uidByte[1] == MASTER_1 &&
      rfid.uid.uidByte[2] == MASTER_2 &&
      rfid.uid.uidByte[3] == MASTER_3) {
    Serial.println("Access Authorized! Unlocking door (90 degrees)...");
    myServo.write(90);
    
    delay(5000); // Hold door unlocked for 5 seconds
    
    Serial.println("Locking door (0 degrees)...");
    myServo.write(0);
  } else {
    Serial.println("Access Denied! Door remains locked.");
  }
  
  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **MFRC522 RFID Reader**, and **Servo Motor** onto the canvas.
2. Wire the MFRC522 SPI lines as listed. Connect the Servo Signal to **GPIO 13**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.
6. Drag a card tag to the reader widget. If its UID is `83 A2 C3 E4` the Servo rotates 90 degrees, pauses for 5 seconds, then rotates back to 0 degrees.

## Expected Output
Serial Monitor:
```
RFID Servo Door Lock active. Scan Card...
Scanned UID: 83 A2 C3 E4
Access Authorized! Unlocking door (90 degrees)...
Locking door (0 degrees)...
Scanned UID: 12 34 56 78
Access Denied! Door remains locked.
```

## Expected Canvas Behavior
* Scanning the matching card tag causes the Servo Motor's arm to rotate to 90 degrees instantly.
* The servo remains at 90 degrees for 5 seconds, then rotates back to 0 degrees.
* Scanning any non-matching card tag produces no movement from the servo.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `myServo.attach(13)` | Configures GPIO 13 as a PWM output for servo control. |
| `myServo.write(0)` | Initializes the servo position to 0 degrees (locked state). |
| `myServo.write(90)` | Rotates the servo position to 90 degrees (unlocked state). |
| `delay(5000)` | Keeps the servo in the unlocked state for 5 seconds before re-locking. |

## Hardware & Safety Concept
* **Actuator Control Isolation**: Servo motors draw significant current spikes during movement. Powering a servo directly from a microcontroller's logic outputs can cause current overload and system resets. Always ensure that the servo is connected to a dedicated 5V power supply line (like the ARIES 5V rail), and share grounds to prevent reference voltage shifts.

## Try This! (Challenges)
1. **Access Indicator Chime**: Add an active buzzer on GPIO 14 that play a short chime when the door is unlocked and a warning tone when re-locked.
2. **Push Button Override**: Connect a Push Button on GPIO 16. Modify the code so that pressing the button unlocks the door for 5 seconds even without scanning a card.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo does not rotate | Incorrect signal pin | Verify the servo signal wire connects to GPIO 13. |
| Servo jitters or system resets | Current overload | Ensure the servo VCC is connected to the 5V rail. |
| Servo rotates to wrong angles | Calibration mismatch | Calibrate the min/max pulse widths in `myServo.attach(13, min, max)` if needed. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [84 - MFRC522 RFID Card UID Serial Logs](84-mfrc522-rfid-card-uid-serial.md)
- [85 - MFRC522 RFID Keycard Access LED](85-mfrc522-rfid-keycard-access-led.md)
- [51 - Servo Motor 180 Degree Sweep](51-servo-motor-180-degree-sweep.md)
