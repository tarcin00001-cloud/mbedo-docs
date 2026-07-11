# 165 - Smart Door Lock with PIR Auto-Relock

Unlock a door using an MFRC522 RFID reader, monitor post-entry activity with a PIR motion sensor on GPIO 17, and automatically relock the servo on GPIO 13 after a configurable timeout when no further motion is detected.

## Goal
Learn to build a multi-stage access control state machine that combines RFID authentication, servo-based locking, PIR-triggered entry detection, and timed auto-relock. Understand how to transition between LOCKED, UNLOCKED, ENTRY DETECTED, and RELOCKING states without using loops.

## What You Will Build
An MFRC522 RFID module is connected via SPI. When a registered card is presented, the servo on GPIO 13 rotates to the unlocked position. The PIR sensor on GPIO 17 monitors for entry; once motion is detected and then ceases for a set timeout (5 seconds), the servo relocks. If no entry is detected within 10 seconds of unlocking, the lock also automatically re-engages. Status messages are printed to the Serial Monitor throughout.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MFRC522 RFID Module | `mfrc522` | Yes | Yes |
| PIR Motion Sensor (HC-SR501) | `pir` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| Green LED (Unlocked indicator) | `led` | Yes | Yes |
| Red LED (Locked indicator) | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 Module | VCC | 3V3 | Red | 3.3 V supply only |
| MFRC522 Module | GND | GND | Black | Common ground |
| MFRC522 Module | SCK | GPIO 2 (SPI SCK) | Yellow | SPI clock |
| MFRC522 Module | MOSI | GPIO 3 (SPI MOSI) | Blue | SPI master-out |
| MFRC522 Module | MISO | GPIO 4 (SPI MISO) | Green | SPI master-in |
| MFRC522 Module | SDA/CS | GPIO 5 (SPI CS) | Orange | SPI chip select |
| MFRC522 Module | RST | GPIO 6 | White | Hardware reset |
| PIR Sensor | VCC | 5V | Red | Sensor power |
| PIR Sensor | GND | GND | Black | Common ground |
| PIR Sensor | OUT | GPIO 17 | Yellow | Motion output |
| Servo | Signal | GPIO 13 | White | PWM lock/unlock |
| Servo | VCC | 5V | Red | Servo power |
| Servo | GND | GND | Black | Servo ground |
| Green LED | Anode | GPIO 24 (LED_G) | Green | Unlocked indicator |
| Green LED | Cathode | GND via 330 Ω | — | Current limiting |
| Red LED | Anode | GPIO 23 (LED_R) | Red | Locked indicator |
| Red LED | Cathode | GND via 330 Ω | — | Current limiting |

> **Wiring tip:** The MFRC522 operates on 3.3 V only; connecting VCC to 5 V will permanently damage the module. Ensure the SPI CS and RST pin numbers in the code match your physical wiring. The PIR sensor has a warm-up time of approximately 30–60 seconds after power-on before it reliably detects motion; allow this delay during hardware testing.

## Code
```cpp
// Smart Door Lock with PIR Auto-Relock
// RFID: SPI (CS=GPIO5, RST=GPIO6)
// PIR: GPIO 17  |  Servo lock: GPIO 13

#include <SPI.h>
#include <MFRC522.h>
#include <Servo.h>

#define PIR_PIN     17
#define SERVO_PIN   13
#define SS_PIN       5
#define RST_PIN      6

// Servo angles
const int LOCKED_ANGLE   = 0;
const int UNLOCKED_ANGLE = 90;

// Timeout periods in milliseconds
const long ENTRY_TIMEOUT_MS  = 10000L;  // Relock if no PIR within 10 s of unlock
const long RELOCK_DELAY_MS   =  5000L;  // Relock 5 s after motion stops

// Authorised card UID (update with your card's UID)
const byte AUTH_UID[4] = {0xDE, 0xAD, 0xBE, 0xEF};

MFRC522 rfid(SS_PIN, RST_PIN);
Servo   lockServo;

// State machine states
// 0 = LOCKED, 1 = UNLOCKED (waiting entry), 2 = ENTRY_DETECTED, 3 = RELOCKING
int   lockState      = 0;
long  unlockTimer    = 0L;   // ms since unlock
long  motionStopTimer = 0L;  // ms since motion last seen
int   lastPir        = LOW;
int   curPir         = LOW;

void setup() {
  Serial.begin(115200);
  SPI.begin();
  rfid.PCD_Init();
  lockServo.attach(SERVO_PIN);
  pinMode(PIR_PIN, INPUT);
  pinMode(LED_R, OUTPUT);
  pinMode(LED_G, OUTPUT);

  lockServo.write(LOCKED_ANGLE);
  digitalWrite(LED_R, HIGH);
  digitalWrite(LED_G, LOW);

  Serial.println("=== Smart Door Lock ===");
  Serial.println("State: LOCKED — Present RFID card to unlock.");
}

void loop() {
  curPir = digitalRead(PIR_PIN);

  // --- STATE 0: LOCKED — waiting for valid RFID card ---
  if (lockState == 0) {
    if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
      bool authorised = true;
      if (rfid.uid.size != 4) { authorised = false; }
      if (authorised && rfid.uid.uidByte[0] != AUTH_UID[0]) { authorised = false; }
      if (authorised && rfid.uid.uidByte[1] != AUTH_UID[1]) { authorised = false; }
      if (authorised && rfid.uid.uidByte[2] != AUTH_UID[2]) { authorised = false; }
      if (authorised && rfid.uid.uidByte[3] != AUTH_UID[3]) { authorised = false; }

      rfid.PICC_HaltA();

      if (authorised) {
        lockServo.write(UNLOCKED_ANGLE);
        digitalWrite(LED_R, LOW);
        digitalWrite(LED_G, HIGH);
        lockState    = 1;
        unlockTimer  = 0L;
        Serial.println("RFID OK — State: UNLOCKED. Waiting for entry...");
      } else {
        Serial.println("RFID DENIED — card not authorised.");
      }
    }
    return;
  }

  // --- STATE 1: UNLOCKED — waiting for PIR entry detection ---
  if (lockState == 1) {
    unlockTimer++;
    if (curPir == HIGH) {
      lockState       = 2;
      motionStopTimer = 0L;
      Serial.println("Entry detected — State: ENTRY_DETECTED. Monitoring...");
    } else if (unlockTimer >= ENTRY_TIMEOUT_MS) {
      lockState = 0;
      lockServo.write(LOCKED_ANGLE);
      digitalWrite(LED_R, HIGH);
      digitalWrite(LED_G, LOW);
      Serial.println("No entry timeout — State: LOCKED.");
    }
    delay(1);
    return;
  }

  // --- STATE 2: ENTRY_DETECTED — monitor until motion stops ---
  if (lockState == 2) {
    if (curPir == HIGH) {
      motionStopTimer = 0L;   // Reset relock timer while motion detected
    } else {
      motionStopTimer++;
      if (motionStopTimer >= RELOCK_DELAY_MS) {
        lockState = 3;
        Serial.println("Motion stopped — State: RELOCKING...");
      }
    }
    delay(1);
    return;
  }

  // --- STATE 3: RELOCKING ---
  if (lockState == 3) {
    lockServo.write(LOCKED_ANGLE);
    digitalWrite(LED_R, HIGH);
    digitalWrite(LED_G, LOW);
    lockState = 0;
    Serial.println("State: LOCKED — Door relocked.");
    return;
  }

  lastPir = curPir;
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board onto the canvas.
2. Drag an **MFRC522** component; connect SPI pins as documented in the wiring table.
3. Drag a **PIR Sensor** component; connect OUT to **GPIO 17**.
4. Drag a **Servo** component; connect Signal to **GPIO 13**.
5. Paste the code into the editor.
6. Select **Interpreted Mode** from the simulation dropdown.
7. Click **Run**.
8. Click **Scan Card** on the MFRC522 widget. If the simulated UID matches `AUTH_UID`, the servo swings to 90° and the green LED lights.
9. Toggle the PIR widget HIGH to simulate entry; after toggling it LOW for 5 seconds, the servo returns to 0° and the red LED lights.

## Expected Output
Serial Monitor:
```
=== Smart Door Lock ===
State: LOCKED — Present RFID card to unlock.
RFID OK — State: UNLOCKED. Waiting for entry...
Entry detected — State: ENTRY_DETECTED. Monitoring...
Motion stopped — State: RELOCKING...
State: LOCKED — Door relocked.
```

## Expected Canvas Behavior
* The servo widget shows 0° (locked) at startup, with the red LED lit.
* On a successful card scan, the servo moves to 90° (unlocked) and the green LED turns on.
* When the PIR is toggled HIGH (entry), the state advances to ENTRY_DETECTED.
* After the PIR goes LOW for 5 seconds, the servo returns to 0° and the red LED turns on.
* If no PIR activity occurs within 10 seconds of unlock, the lock re-engages automatically.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `#include <MFRC522.h>` | Imports the MFRC522 SPI RFID library. |
| `AUTH_UID[4] = {0xDE, 0xAD, 0xBE, 0xEF}` | Defines the authorised card UID as four hex bytes. |
| `rfid.PICC_IsNewCardPresent()` | Checks if a new card has entered the reader field. |
| `rfid.PICC_ReadCardSerial()` | Reads the card's UID into `rfid.uid`. |
| `rfid.uid.uidByte[n] != AUTH_UID[n]` | Compares each UID byte sequentially without array iteration. |
| `lockState = 1` | Transitions the state machine to the UNLOCKED state. |
| `unlockTimer++; delay(1)` | Counts milliseconds elapsed since unlock without a blocking sleep. |
| `motionStopTimer >= RELOCK_DELAY_MS` | Triggers relocking after 5 s of continuous no-motion. |
| `lockServo.write(LOCKED_ANGLE)` | Returns the servo to 0° (locked position). |

## Hardware & Safety Concept
* **State Machine Design**: The system uses four discrete states rather than nested conditions or blocking delays. Each state has a clear entry condition, action, and transition rule. This pattern is essential for real-time embedded systems where blocking calls would prevent the board from responding to sensors.
* **RFID Security Note**: The UID stored in `AUTH_UID` is transmitted in plain text over SPI. For a production system, store multiple authorised UIDs in EEPROM and implement a hash-based comparison. The MFRC522 also supports Mifare DES/AES-encrypted authentication for stronger security.
* **PIR Warm-Up**: The HC-SR501 PIR sensor has an internal calibration routine that takes 30–60 seconds after power-on. During this window it may produce spurious HIGH pulses. Allow for this delay in hardware deployments by ignoring PIR readings during the first 60 seconds of operation.

## Try This! (Challenges)
1. **Multiple Authorised Cards**: Store up to three different UIDs in separate `byte` variables (no arrays) and check each one sequentially inside the state machine's LOCKED state, granting access if any one matches.
2. **Wrong Card Alert**: Flash the onboard red LED (LED_R, GPIO 23) three times and sound the buzzer (GPIO 14) for 500 ms when an unauthorised card is presented, providing a clear rejection feedback.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| RFID never responds | SPI wiring incorrect or MFRC522 VCC at 5V | Verify SPI pins match SS_PIN/RST_PIN; MFRC522 requires 3.3 V — not 5 V. |
| Card always denied | AUTH_UID bytes do not match card | Temporarily print `rfid.uid.uidByte[0..3]` to Serial, copy the actual UID into the code. |
| Servo does not move | Servo library not attached or wrong pin | Confirm `lockServo.attach(SERVO_PIN)` is in `setup()` and SERVO_PIN = 13. |
| Auto-relock never triggers | PIR stays HIGH indefinitely | In simulation, manually toggle PIR LOW; in hardware, adjust the HC-SR501 sensitivity potentiometer. |
| Lock re-engages immediately after unlock | ENTRY_TIMEOUT_MS too small | Increase `ENTRY_TIMEOUT_MS` to 15000 (15 seconds). |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [164 - Water Flow Rate Station](164-water-flow-rate-station.md)
- [168 - Bluetooth Keypad Entry Lock](168-bluetooth-keypad-entry-lock.md)
- [166 - Smart Fan with Temperature Hysteresis](166-smart-fan.md)
