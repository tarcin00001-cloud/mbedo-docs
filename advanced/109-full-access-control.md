# 109 - Full Access Control

Build a complete RFID access control system that drives a servo, a green LED, a red LED, and a buzzer — authorized cards open the gate with a double-beep; unauthorized cards trigger a long alarm tone.

## Goal
Learn how to combine RFID UID authentication with multiple simultaneous output actuators (servo, dual LEDs, buzzer) to create a realistic, hardware-complete access control system without custom helper functions.

## What You Will Build
When a card is scanned:

- **Authorized card**: Servo sweeps to 90 degrees (gate opens), green LED lights for 3 seconds, buzzer plays two ascending beeps, then everything returns to the locked state.
- **Unauthorized card**: Servo stays at 0 degrees (locked), red LED lights, buzzer plays one long 600 ms alarm tone, then red LED turns off.

The scan result also prints to the Serial Terminal on every event.

**Why D3, D5, D6, D8, D9, D10–D13?** The MFRC522 uses the SPI bus (D10 CS, D11 MOSI, D12 MISO, D13 SCK) plus D9 RST. Pin D6 drives the servo signal (PWM). Pins D3 and D5 drive the green and red LEDs. Pin D8 drives the buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes |
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
| Servo Motor | Signal | D6 | PWM control |
| Servo Motor | VCC | 5V | Power supply |
| Servo Motor | GND | GND | Ground reference |
| Green LED | Anode (+) | D3 | Access granted indicator |
| Green LED | Cathode (-) | GND | Via 220 ohm resistor |
| Red LED | Anode (+) | D5 | Access denied indicator |
| Red LED | Cathode (-) | GND | Via 220 ohm resistor |
| Buzzer | + | D8 | Signal pin |
| Buzzer | - | GND | Ground reference |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>
#include <Servo.h>

const int RST_PIN   = 9;
const int SS_PIN    = 10;
const int SERVO_PIN = 6;
const int GREEN_PIN = 3;
const int RED_PIN   = 5;
const int BUZ_PIN   = 8;

// Authorized card UID (4 bytes) - update to match your card
const byte AUTH_B0 = 0xDE;
const byte AUTH_B1 = 0xAD;
const byte AUTH_B2 = 0xBE;
const byte AUTH_B3 = 0xEF;

MFRC522 rfid(SS_PIN, RST_PIN);
Servo gateServo;

void setup() {
  Serial.begin(9600);
  SPI.begin();
  rfid.PCD_Init();

  gateServo.attach(SERVO_PIN);
  gateServo.write(0); // Start locked

  pinMode(GREEN_PIN, OUTPUT);
  pinMode(RED_PIN,   OUTPUT);
  pinMode(BUZ_PIN,   OUTPUT);

  digitalWrite(GREEN_PIN, LOW);
  digitalWrite(RED_PIN,   LOW);
  noTone(BUZ_PIN);

  Serial.println("Full Access Control Ready. Scan a card...");
}

void loop() {
  if (!rfid.PICC_IsNewCardPresent() || !rfid.PICC_ReadCardSerial()) return;

  // Print scanned UID (fixed 4 bytes for standard Mifare cards)
  Serial.print("UID: ");
  Serial.print(rfid.uid.uidByte[0], HEX); Serial.print(" ");
  Serial.print(rfid.uid.uidByte[1], HEX); Serial.print(" ");
  Serial.print(rfid.uid.uidByte[2], HEX); Serial.print(" ");
  Serial.println(rfid.uid.uidByte[3], HEX);

  // Compare UID byte-by-byte (interpreted mode safe — no arrays or loops)
  if (rfid.uid.uidByte[0] == AUTH_B0 &&
      rfid.uid.uidByte[1] == AUTH_B1 &&
      rfid.uid.uidByte[2] == AUTH_B2 &&
      rfid.uid.uidByte[3] == AUTH_B3) {

    // ACCESS GRANTED
    Serial.println("-> ACCESS GRANTED");
    digitalWrite(GREEN_PIN, HIGH);
    gateServo.write(90); // Open gate

    tone(BUZ_PIN, 1000, 100); delay(200);
    tone(BUZ_PIN, 1400, 100); delay(300);
    noTone(BUZ_PIN);

    delay(3000); // Hold open

    gateServo.write(0); // Close gate
    digitalWrite(GREEN_PIN, LOW);
    Serial.println("-> Gate closed.");

  } else {

    // ACCESS DENIED
    Serial.println("-> ACCESS DENIED");
    digitalWrite(RED_PIN, HIGH);
    tone(BUZ_PIN, 350, 600);
    delay(700);
    noTone(BUZ_PIN);
    digitalWrite(RED_PIN, LOW);
  }

  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
  delay(500);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **MFRC522 RFID Reader**, **Servo Motor**, **Green LED**, **Red LED**, and **Buzzer** onto the canvas.
2. Wire MFRC522: **SDA** to **D10**, **SCK** to **D13**, **MOSI** to **D11**, **MISO** to **D12**, **RST** to **D9**, **3.3V** to **3.3V**, **GND** to **GND**.
3. Connect Servo: **Signal** to **D6**, **VCC** to **5V**, **GND** to **GND**.
4. Connect Green LED anode to **D3**, cathode to **GND**.
5. Connect Red LED anode to **D5**, cathode to **GND**.
6. Connect Buzzer **+** to **D8**, **-** to **GND**.
7. Paste the code into the editor.
8. Use the default Arduino interpreted runtime.
9. Click **Run**.
10. Click the RFID reader to simulate card scans and observe the servo angle, LED states, and buzzer tones.

## Expected Output

Terminal (Authorized card):
```
Full Access Control Ready. Scan a card...
UID: DE AD BE EF
-> ACCESS GRANTED
-> Gate closed.
```

Terminal (Unknown card):
```
UID: AA BB CC DD
-> ACCESS DENIED
```

## Expected Canvas Behavior
| Scanned UID | Match | Servo | Green LED | Red LED | Buzzer |
| --- | --- | --- | --- | --- | --- |
| `DE AD BE EF` | Yes | 0 → 90 → 0 deg | ON 3 s | OFF | Two ascending beeps |
| `AA BB CC DD` | No | 0 deg (no change) | OFF | ON 700 ms | One 600 ms alarm tone |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `const byte AUTH_B0 … AUTH_B3` | Stores each byte of the authorized UID as a separate constant — avoids arrays and keeps code interpreter-safe. |
| `rfid.uid.uidByte[0] == AUTH_B0 && …` | Compares all four UID bytes in a single boolean expression without a loop. |
| `gateServo.write(90)` | Immediately jumps the servo to the open position; MbedO simulates the motion on the canvas. |
| `tone(BUZ_PIN, 1000, 100); delay(200)` | Plays a 100 ms tone then waits 200 ms before the next, creating a two-tone ascending chime effect. |
| `rfid.PICC_HaltA()` | Puts the card into HALT state so the same card is not re-read continuously on the next loop iteration. |

## Hardware & Safety Concept: Fail-Safe Locked Default
The servo is initialized to 0 degrees in `setup()`. If the Arduino resets, loses power, or encounters a firmware error, the gate returns to the physically locked position rather than remaining open. This "fail-safe closed" design is standard practice in all access control hardware, from electromagnetic door strikes to turnstile barriers.

## Try This! (Challenges)
1. **Second Authorized Card**: Add a second set of `AUTH2_B0`…`AUTH2_B3` constants and use an `else if` block to grant access to a second card with a different UID.
2. **Lockout Counter**: Add a global `int deniedCount = 0` variable. Increment it on each denial, and if it reaches 3, activate a longer alarm and print `LOCKOUT` to the Terminal.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Card never detected | SPI wiring error | Verify all five SPI lines: CS → D10, MOSI → D11, MISO → D12, SCK → D13, RST → D9. |
| Authorized card triggers DENIED | UID bytes do not match | Scan your card, read the printed UID from Terminal, and update the four `AUTH_Bx` constants. |
| Servo does not move | Wrong pin or no `attach()` | Confirm `gateServo.attach(SERVO_PIN)` is called in `setup()` before any `write()`. |

## Mode Notes
RFID SPI reads, byte-by-byte UID comparison, servo writes, and tone output are all supported by MbedO interpreted mode.

## Related Projects
- [70 - RFID Servo Unlock](../intermediate/70-rfid-servo-unlock.md)
- [93 - RFID LED Buzzer](93-rfid-led-buzzer.md)
- [94 - RFID Servo LCD](94-rfid-servo-lcd.md)
- [110 - RFID LCD Status](110-rfid-lcd-status.md)
