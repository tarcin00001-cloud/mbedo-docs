# 124 - Pico Smart Lock

Build an advanced high-security vault door lock requiring both authorized RFID cards and a 4-digit keypad PIN to open.

## Goal
Learn how to implement multi-factor authentication (MFA) combining SPI RFID readers, multi-pin matrix keypad scanners (without loops or arrays), and servo motors.

## What You Will Build
A dual-authentication secure lock:
- **MFRC522 RFID Reader**: Reads user tags.
- **4x4 Matrix Keypad (GP2, GP3, GP6, GP7, GP8, GP9, GP20, GP21)**: Captures user PIN input.
- **Servo Motor (GP10)**: Swings 90° to unlock the door when both card and PIN (`1234`) are validated.
- **16x2 I2C LCD**: Displays instructions (e.g. "Scan Card...", "Enter PIN: ****").
- **Active Buzzer (GP14)**: Plays confirmation tone patterns.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MFRC522 | SDA / SCK / MOSI / MISO / RST | GP17 / GP18 / GP19 / GP16 / GP22 | SPI Bus |
| Keypad | Row 1 / Row 2 / Row 3 / Row 4 | GP2 / GP3 / GP6 / GP7 | Output scanning rows |
| Keypad | Col 1 / Col 2 / Col 3 / Col 4 | GP8 / GP9 / GP20 / GP21 | Input columns (pull-up) |
| Servo Motor | Signal / VCC / GND | GP10 / VBUS (5V) / GND | Lock actuator |
| Active Buzzer | VCC / GND | GP14 / GND | Sounder |
| I2C LCD | SDA / SCL | GP4 / GP5 | Status screen |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Servo.h>

const int RST_PIN   = 22;
const int SS_PIN    = 17;
const int SERVO_PIN = 10;
const int BUZZ_PIN  = 14;

// Keypad pin mappings
const int R1 = 2; const int R2 = 3; const int R3 = 6; const int R4 = 7;
const int C1 = 8; const int C2 = 9; const int C3 = 20; const int C4 = 21;

MFRC522 mfrc522(SS_PIN, RST_PIN);
LiquidCrystal_I2C lcd(0x27, 16, 2);
Servo lockServo;

const byte authUID[4] = {0xDE, 0x12, 0xA4, 0xC3};
bool cardValidated = false;

int pinCount = 0;
char enteredPIN[5] = "    "; // Store 4 digits + null terminator
char targetPIN[5]  = "1234";

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  SPI.begin();
  mfrc522.PCD_Init();
  
  lockServo.attach(SERVO_PIN);
  lockServo.write(0); // Start locked

  pinMode(BUZZ_PIN, OUTPUT);
  digitalWrite(BUZZ_PIN, LOW);

  // Setup keypad pins
  pinMode(R1, OUTPUT); pinMode(R2, OUTPUT); pinMode(R3, OUTPUT); pinMode(R4, OUTPUT);
  pinMode(C1, INPUT_PULLUP); pinMode(C2, INPUT_PULLUP); pinMode(C3, INPUT_PULLUP); pinMode(C4, INPUT_PULLUP);

  lcd.init();
  lcd.backlight();
  resetLockState();
}

void loop() {
  // Phase 1: RFID Verification
  if (!cardValidated) {
    if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
      bool match = true;
      if (mfrc522.uid.uidByte[0] != authUID[0]) { match = false; }
      if (mfrc522.uid.uidByte[1] != authUID[1]) { match = false; }
      if (mfrc522.uid.uidByte[2] != authUID[2]) { match = false; }
      if (mfrc522.uid.uidByte[3] != authUID[3]) { match = false; }

      if (match) {
        cardValidated = true;
        tone(BUZZ_PIN, 1000, 200);
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Card Approved");
        lcd.setCursor(0, 1);
        lcd.print("Enter PIN: ");
      } else {
        tone(BUZZ_PIN, 300, 600);
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Access Denied!");
        delay(1500);
        resetLockState();
      }
      mfrc522.PICC_HaltA();
    }
  } 
  // Phase 2: Keypad Entry
  else {
    char key = scanKeypad();
    if (key != '\0') {
      tone(BUZZ_PIN, 800, 80);
      
      // Store digit
      enteredPIN[pinCount] = key;
      pinCount = pinCount + 1;
      
      lcd.setCursor(11 + pinCount - 1, 1);
      lcd.print("*"); // Hide actual characters for security

      delay(300); // Debounce key press

      if (pinCount >= 4) {
        // Compare PIN
        bool pinMatch = true;
        if (enteredPIN[0] != targetPIN[0]) { pinMatch = false; }
        if (enteredPIN[1] != targetPIN[1]) { pinMatch = false; }
        if (enteredPIN[2] != targetPIN[2]) { pinMatch = false; }
        if (enteredPIN[3] != targetPIN[3]) { pinMatch = false; }

        lcd.clear();
        if (pinMatch) {
          lcd.setCursor(0, 0);
          lcd.print("VAULT UNLOCKED");
          lockServo.write(90); // Unlock
          tone(BUZZ_PIN, 1200, 500);
          delay(4000); // Keep open for 4 seconds
        } else {
          lcd.setCursor(0, 0);
          lcd.print("Incorrect PIN!");
          tone(BUZZ_PIN, 200, 800);
          delay(2000);
        }
        resetLockState();
      }
    }
  }

  delay(50);
}

void resetLockState() {
  cardValidated = false;
  pinCount = 0;
  enteredPIN[0] = ' '; enteredPIN[1] = ' '; enteredPIN[2] = ' '; enteredPIN[3] = ' ';
  lockServo.write(0); // Lock door
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Vault Locked");
  lcd.setCursor(0, 1);
  lcd.print("Scan RFID Card..");
}

// Low-level matrix keypad scanner avoiding loops and array definitions
char scanKeypad() {
  char pressedKey = '\0';

  // Scan Row 1
  digitalWrite(R1, LOW); digitalWrite(R2, HIGH); digitalWrite(R3, HIGH); digitalWrite(R4, HIGH);
  if (digitalRead(C1) == LOW) { pressedKey = '1'; }
  if (digitalRead(C2) == LOW) { pressedKey = '2'; }
  if (digitalRead(C3) == LOW) { pressedKey = '3'; }
  if (digitalRead(C4) == LOW) { pressedKey = 'A'; }

  // Scan Row 2
  digitalWrite(R1, HIGH); digitalWrite(R2, LOW); digitalWrite(R3, HIGH); digitalWrite(R4, HIGH);
  if (digitalRead(C1) == LOW) { pressedKey = '4'; }
  if (digitalRead(C2) == LOW) { pressedKey = '5'; }
  if (digitalRead(C3) == LOW) { pressedKey = '6'; }
  if (digitalRead(C4) == LOW) { pressedKey = 'B'; }

  // Scan Row 3
  digitalWrite(R1, HIGH); digitalWrite(R2, HIGH); digitalWrite(R3, LOW); digitalWrite(R4, HIGH);
  if (digitalRead(C1) == LOW) { pressedKey = '7'; }
  if (digitalRead(C2) == LOW) { pressedKey = '8'; }
  if (digitalRead(C3) == LOW) { pressedKey = '9'; }
  if (digitalRead(C4) == LOW) { pressedKey = 'C'; }

  // Scan Row 4
  digitalWrite(R1, HIGH); digitalWrite(R2, HIGH); digitalWrite(R3, HIGH); digitalWrite(R4, LOW);
  if (digitalRead(C1) == LOW) { pressedKey = '*'; }
  if (digitalRead(C2) == LOW) { pressedKey = '0'; }
  if (digitalRead(C3) == LOW) { pressedKey = '#'; }
  if (digitalRead(C4) == LOW) { pressedKey = 'D'; }

  return pressedKey;
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MFRC522 RFID Reader**, **4x4 Keypad**, **Servo**, **Active Buzzer**, and **I2C LCD** onto the canvas.
2. Connect MFRC522 SPI pins, Keypad Row/Col pins, Servo to **GP10**, Buzzer to **GP14**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Scan the authorized RFID card, then type the PIN code `1234` on the keypad to swing the lock open.

## Expected Output

Terminal:
```
Simulation active. Multi-factor authentication lock online.
```

## Expected Canvas Behavior
* Startup: LCD reads `Vault Locked` / `Scan RFID Card..`. Servo at 0°.
* Scan authorized card: LCD changes to `Card Approved` / `Enter PIN: `.
* Press `1`, `2`, `3`, `4`: LCD appends `*` character for each press. After the fourth key, LCD reads `VAULT UNLOCKED`, Servo sweeps to 90° for 4 seconds, then returns to 0° and resets.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `cardValidated = true` | Toggles the authentication phase tracker from card verification to keypad PIN entry. |
| `scanKeypad()` | Runs a low-level sequential row-scanning block to read key coordinates without violating loop/array rules. |

## Hardware & Safety Concept: Multi-Factor Authentication
In security architecture, Multi-Factor Authentication (MFA) requires presenting credentials from at least two different categories:
1. **Something You Have**: An RFID keycard, hardware token, or USB key.
2. **Something You Know**: A password, PIN code, or secret answer.
3. **Something You Are**: Biometrics (fingerprint, iris scan).
Combining a physical card and a keypad PIN ensures the vault stays secure even if a keycard is lost or stolen.

## Try This! (Challenges)
1. **Entry Limit**: Sound a continuous alarm if an incorrect PIN is entered three times in a row.
2. **Clear Key**: Make the `'*'` key clear the current PIN input so the user can re-type it.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Keypad inputs register multiple times | Bounce delay too low | Increase the debounce delay (`delay(300)`) after key detection to let the key contact fully settle. |

## Mode Notes
This multi-device I2C, SPI, and multiplexed project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [87 - Pico RFID Servo](../intermediate/87-pico-rfid-servo.md)
- [111 - Pico RFID LCD](../intermediate/111-pico-rfid-lcd.md)
