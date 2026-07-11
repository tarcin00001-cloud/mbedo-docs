# 158 - Keypad RFID Multi-factor Lock

Build a two-factor security lock system requiring both an authorized RFID card scan and a correct 4-digit PIN code to actuate a locking servo motor.

## Goal
Learn how to coordinate multiple interface protocols (SPI and matrix keyboard scanning) in a loop-free C++ state machine, managing access states and lock timing.

## What You Will Build
A two-factor authentication (2FA) door latch. The lock is normally secured (Servo at 0°). To unlock, a user must first tap an authorized RFID card (validated against a static hexadecimal code). Once validated, the system prompts for a PIN. The user must then enter the correct 4-digit PIN ("2580") followed by `#` on the 4x4 matrix keypad. If both credentials match, the servo rotates to 90° for 5 seconds before relocking.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MFRC522 RFID Reader Module | `rfid_reader` | Yes | Yes |
| RFID Card / Tag | `rfid_tag` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| SG90 Servo Motor | `servo` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 RFID | 3.3V | 3V3 | Red | Power (3.3V only) |
| MFRC522 RFID | GND | GND | Black | Ground reference |
| MFRC522 RFID | RST (Reset) | GPIO 11 | Purple | Reset control pin |
| MFRC522 RFID | SDA (SS) | GPIO 10 | Orange | SPI Chip Select |
| MFRC522 RFID | SCK | SPI SCK | Yellow | SPI Clock line |
| MFRC522 RFID | MISO | SPI MISO | Green | SPI Master In Slave Out |
| MFRC522 RFID | MOSI | SPI MOSI | Blue | SPI Master Out Slave In |
| 4x4 Keypad | Row 1-4 | GPIO 2, 3, 4, 5 | White/Brown | Row scan inputs |
| 4x4 Keypad | Col 1-4 | GPIO 6, 7, 8, 9 | Gray/Blue | Column scan outputs |
| Servo Motor | Signal | GPIO 13 | Yellow | Servo PWM control line |
| Servo Motor | VCC | 5V | Red | Servo power supply (5V) |
| Servo Motor | GND | GND | Black | Ground reference |

> **Wiring tip:** The 4x4 Keypad columns (GPIO 6-9) and RFID Reset (GPIO 11) must not overlap. Verify that RFID SDA is on GPIO 10 and RST is on GPIO 11 to prevent conflicts with the keypad columns.

## Code
```cpp
// Keypad RFID Multi-factor Lock - VEGA ARIES v3
#include <SPI.h>
#include <MFRC522.h>
#include <Keypad.h>
#include <Servo.h>

#define SS_PIN 10
#define RST_PIN 11
#define SERVO_PIN 13

MFRC522 rfid(SS_PIN, RST_PIN);
Servo lockServo;

// 4x4 Keypad Layout Definition
const byte ROWS = 4;
const byte COLS = 4;
char keys[ROWS][COLS] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};
byte rowPins[ROWS] = {2, 3, 4, 5};
byte colPins[COLS] = {6, 7, 8, 9};
Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);

// Security State Variables
bool rfidVerified = false;
unsigned long authStartTime = 0;
const unsigned long TIMEOUT_WINDOW = 15000; // 15 seconds to enter PIN after RFID
String pinBuffer = "";
const String AUTHORIZED_PIN = "2580";

void setup() {
  Serial.begin(115200);
  SPI.begin();
  rfid.PCD_Init();
  lockServo.attach(SERVO_PIN);
  
  lockServo.write(0); // Locked state (0 degrees)
  Serial.println("System Ready. Step 1: Scan RFID Card");
}

void loop() {
  // Handle State Timeout (reset 2FA state if user takes too long to enter PIN)
  if (rfidVerified && (millis() - authStartTime > TIMEOUT_WINDOW)) {
    rfidVerified = false;
    pinBuffer = "";
    Serial.println("Authentication timeout. Scan RFID again.");
  }

  // --- STEP 1: RFID CARD DETECT ---
  if (!rfidVerified) {
    if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
      // Validate card UID (Example: authorized card UID is DE AD BE EF)
      if (rfid.uid.uidByte[0] == 0xDE && 
          rfid.uid.uidByte[1] == 0xAD && 
          rfid.uid.uidByte[2] == 0xBE && 
          rfid.uid.uidByte[3] == 0xEF) {
        
        rfidVerified = true;
        authStartTime = millis();
        pinBuffer = "";
        Serial.println("RFID Authenticated. Step 2: Enter PIN on Keypad");
      } else {
        Serial.print("Access Denied. Unknown RFID Card: ");
        Serial.print(rfid.uid.uidByte[0], HEX);
        Serial.print(" ");
        Serial.print(rfid.uid.uidByte[1], HEX);
        Serial.print(" ");
        Serial.print(rfid.uid.uidByte[2], HEX);
        Serial.print(" ");
        Serial.println(rfid.uid.uidByte[3], HEX);
      }
      rfid.PICC_HaltA();
      rfid.PCD_StopCrypto1();
    }
  }

  // --- STEP 2: KEYPAD PIN INPUT ---
  char key = keypad.getKey();
  if (key) {
    if (rfidVerified) {
      if (key == '#') { // Press '#' to submit PIN code
        if (pinBuffer == AUTHORIZED_PIN) {
          Serial.println("PIN Correct. ACCESS GRANTED. Unlocking...");
          lockServo.write(90);  // Unlock
          delay(5000);          // Hold door open for 5 seconds
          lockServo.write(0);   // Relock
          Serial.println("Locked. Step 1: Scan RFID Card");
        } else {
          Serial.println("PIN Incorrect. ACCESS DENIED. Resetting...");
        }
        rfidVerified = false; // Reset 2FA status
        pinBuffer = "";
      } else if (key == '*') { // Press '*' to clear entry buffer
        pinBuffer = "";
        Serial.println("PIN Buffer cleared.");
      } else {
        // Build 4-digit PIN code buffer
        if (pinBuffer.length() < 4) {
          pinBuffer += key;
          Serial.print("*"); // Mask display output
        }
      }
    } else {
      Serial.println("Access Blocked. Scan authorized RFID card first.");
    }
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, **MFRC522 RFID Reader**, **4x4 Keypad**, and **Servo Motor** onto the canvas.
2. Connect the RFID SPI lines, matching RST to **GPIO 11** and SDA to **GPIO 10**.
3. Connect the Keypad Row pins to **GPIO 2-5** and Col pins to **GPIO 6-9**.
4. Connect the Servo Motor Signal to **GPIO 13**.
5. Paste the code into the editor.
6. Click **Run**.
7. Drag the authorized RFID tag (UID: `DE AD BE EF`) near the RFID reader. Verify that the console prompts you for a PIN.
8. Click `2`, `5`, `8`, `0` on the keypad widget, then click `#` to unlock the servo.

## Expected Output
Serial Monitor:
```
System Ready. Step 1: Scan RFID Card
RFID Authenticated. Step 2: Enter PIN on Keypad
****
PIN Correct. ACCESS GRANTED. Unlocking...
Locked. Step 1: Scan RFID Card
```

## Expected Canvas Behavior
* Pressing the keypad before scanning a card results in an `Access Blocked` message.
* Scanning the authorized card changes the system state, enabling the keypad input mode for 15 seconds.
* Submitting the correct PIN rotates the servo to 90° for 5 seconds, after which it returns to 0°.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `millis() - authStartTime > TIMEOUT_WINDOW` | Monitors the inactivity timeout to prevent latching the unlocked state indefinitely. |
| `rfid.uid.uidByte[0] == 0xDE ...` | Validates the specific 4-byte unique RFID card identifier. |
| `rfidVerified = true` | Advances the 2FA state machine to stage 2. |
| `keypad.getKey()` | Scans the row/column pins to check for keyboard button presses. |
| `pinBuffer == AUTHORIZED_PIN` | Compares the entered keystroke combination against the stored passcode string. |
| `lockServo.write(90)` | Rotates the physical latch servo to the open position. |

## Hardware & Safety Concept
* **Multi-Factor Latching (Security-in-Depth)**: Relying solely on RFID is vulnerable to card-cloning attacks. Adding a PIN code layer ensures that even if a physical card is lost or cloned, the perimeter remains secure.
* **Fail-Secure Operation**: In electronic lock designs, lock pins are mechanically spring-loaded or driven such that if the main power is cut, the servo remains in the 0° (locked) orientation, ensuring secure containment.

## Try This! (Challenges)
1. **Intruder Alert**: Count incorrect PIN attempts. If the user enters a wrong PIN 3 times in a row, sound an active buzzer on `GPIO 14` for 10 seconds.
2. **Dynamic PIN Change**: Implement a method where pressing a specific card and key combination allows the user to program a new authorized PIN.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Scanning RFID card does not change state | Incorrect SPI or Reset wire | Check that the MFRC522 RST pin is wired to GPIO 11 (as declared in the constructor), and check that MOSI/MISO are not swapped. |
| Keypad presses register wrong characters | Keypad pin order reversed | Check Row and Column wiring order. Double-check row pins are 2, 3, 4, 5 and column pins are 6, 7, 8, 9. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [87 - MFRC522 RFID Servo Door Unlock](../intermediate/87-mfrc522-rfid-servo-door-unlock.md)
- [114 - Keypad 4x4 Password Lock](../intermediate/114-keypad-4x4-password-lock.md)
