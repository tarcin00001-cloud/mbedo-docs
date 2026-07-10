# 157 - Pico Safe Vault EEPROM

Build a secure vault vault alarm that stores user PIN codes in non-volatile flash memory (EEPROM) and supports changing the PIN code via keypad inputs.

## Goal
Learn how to use the `EEPROM` library to read and write multi-character strings (PIN codes), scan matrix keypads, and toggle door relays.

## What You Will Build
An adjustable digital vault door console:
- **Relay Module (GP10)**: Toggles the vault's electronic strike lock.
- **4x4 Keypad**: Receives passcode digits.
- **EEPROM (Flash)**: Stores the 4-digit PIN code (default `1234`) at address `10`.
- **Active Buzzer (GP14)**: Emits beep indicators.
- **16x2 I2C LCD (GP4, GP5)**: Displays status menus (e.g. "PIN Changed" or "Enter Old PIN").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Keypad Rows | R1 - R4 | GP2 / GP3 / GP6 / GP7 | Output scanning rows |
| Keypad Cols | C1 - C4 | GP8 / GP9 / GP20 / GP21 | Input columns (pull-up) |
| Relay Module | IN | GP10 | Solenoid door strike |
| Active Buzzer | VCC (+) | GP14 | Sound chime pin |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <EEPROM.h>

const int RELAY_PIN = 10;
const int BUZZ_PIN  = 14;

// Keypad pins
const int R1 = 2; const int R2 = 3; const int R3 = 6; const int R4 = 7;
const int C1 = 8; const int C2 = 9; const int C3 = 20; const int C4 = 21;

LiquidCrystal_I2C lcd(0x27, 16, 2);

char storedPIN[5]  = "1234";
char enteredPIN[5] = "    ";
int pinCount = 0;
bool changeMode = false;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZ_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(BUZZ_PIN, LOW);

  // Setup keypad pins
  pinMode(R1, OUTPUT); pinMode(R2, OUTPUT); pinMode(R3, OUTPUT); pinMode(R4, OUTPUT);
  pinMode(C1, INPUT_PULLUP); pinMode(C2, INPUT_PULLUP); pinMode(C3, INPUT_PULLUP); pinMode(C4, INPUT_PULLUP);

  EEPROM.begin(512);

  // Load PIN from EEPROM (address 10-13)
  for (int i = 0; i < 4; i++) {
    char val = EEPROM.read(10 + i);
    // If unit initialized, load it, otherwise keep default "1234"
    if (val >= '0' && val <= '9') {
      storedPIN[i] = val;
    }
  }

  lcd.init();
  lcd.backlight();
  showIdleScreen();
}

void loop() {
  char key = scanKeypad();

  if (key != '\0') {
    tone(BUZZ_PIN, 800, 80);
    delay(250); // Key debounce

    if (key == '*') {
      // Enter PIN Change Mode
      changeMode = true;
      pinCount = 0;
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Change PIN Mode");
      lcd.setCursor(0, 1);
      lcd.print("New PIN: ");
    } 
    else if (key == '#') {
      // Clear entry
      pinCount = 0;
      showIdleScreen();
    } 
    else {
      // Record digit
      enteredPIN[pinCount] = key;
      pinCount = pinCount + 1;
      
      lcd.setCursor(9 + pinCount - 1, 1);
      lcd.print("*");

      if (pinCount >= 4) {
        lcd.clear();
        lcd.setCursor(0, 0);

        if (changeMode) {
          // Save new PIN to EEPROM
          storedPIN[0] = enteredPIN[0];
          storedPIN[1] = enteredPIN[1];
          storedPIN[2] = enteredPIN[2];
          storedPIN[3] = enteredPIN[3];

          for (int i = 0; i < 4; i++) {
            EEPROM.write(10 + i, storedPIN[i]);
          }
          EEPROM.commit();

          lcd.print("PIN Updated!");
          lcd.setCursor(0, 1);
          lcd.print("Saved to Flash");
          tone(BUZZ_PIN, 1200, 300);
          delay(2000);
          changeMode = false;
          showIdleScreen();
        } else {
          // Check PIN
          bool match = true;
          if (enteredPIN[0] != storedPIN[0]) { match = false; }
          if (enteredPIN[1] != storedPIN[1]) { match = false; }
          if (enteredPIN[2] != storedPIN[2]) { match = false; }
          if (enteredPIN[3] != storedPIN[3]) { match = false; }

          if (match) {
            lcd.print("Access Granted ");
            lcd.setCursor(0, 1);
            lcd.print("Door Unlocked   ");
            digitalWrite(RELAY_PIN, HIGH); // Open door strike
            tone(BUZZ_PIN, 1000, 500);
            delay(3000);
            digitalWrite(RELAY_PIN, LOW); // Lock door strike
          } else {
            lcd.print("Wrong PIN!      ");
            lcd.setCursor(0, 1);
            lcd.print("Access Denied   ");
            tone(BUZZ_PIN, 200, 800);
            delay(2000);
          }
          showIdleScreen();
        }
      }
    }
  }

  delay(20);
}

void showIdleScreen() {
  pinCount = 0;
  changeMode = false;
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Safe Vault Door");
  lcd.setCursor(0, 1);
  lcd.print("Code: ");
}

char scanKeypad() {
  char pressedKey = '\0';

  // Row 1
  digitalWrite(R1, LOW); digitalWrite(R2, HIGH); digitalWrite(R3, HIGH); digitalWrite(R4, HIGH);
  if (digitalRead(C1) == LOW) { pressedKey = '1'; }
  if (digitalRead(C2) == LOW) { pressedKey = '2'; }
  if (digitalRead(C3) == LOW) { pressedKey = '3'; }
  if (digitalRead(C4) == LOW) { pressedKey = 'A'; }

  // Row 2
  digitalWrite(R1, HIGH); digitalWrite(R2, LOW); digitalWrite(R3, HIGH); digitalWrite(R4, HIGH);
  if (digitalRead(C1) == LOW) { pressedKey = '4'; }
  if (digitalRead(C2) == LOW) { pressedKey = '5'; }
  if (digitalRead(C3) == LOW) { pressedKey = '6'; }
  if (digitalRead(C4) == LOW) { pressedKey = 'B'; }

  // Row 3
  digitalWrite(R1, HIGH); digitalWrite(R2, HIGH); digitalWrite(R3, LOW); digitalWrite(R4, HIGH);
  if (digitalRead(C1) == LOW) { pressedKey = '7'; }
  if (digitalRead(C2) == LOW) { pressedKey = '8'; }
  if (digitalRead(C3) == LOW) { pressedKey = '9'; }
  if (digitalRead(C4) == LOW) { pressedKey = 'C'; }

  // Row 4
  digitalWrite(R1, HIGH); digitalWrite(R2, HIGH); digitalWrite(R3, HIGH); digitalWrite(R4, LOW);
  if (digitalRead(C1) == LOW) { pressedKey = '*'; }
  if (digitalRead(C2) == LOW) { pressedKey = '0'; }
  if (digitalRead(C3) == LOW) { pressedKey = '#'; }
  if (digitalRead(C4) == LOW) { pressedKey = 'D'; }

  return pressedKey;
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **4x4 Keypad**, **Relay**, **Active Buzzer**, and **I2C LCD** onto the canvas.
2. Connect keypad rows/cols, Relay to **GP10**, Buzzer to **GP14**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Type `1234` to unlock the relay. Press `*` followed by `5555` to change the PIN, then reboot the simulation to test the new code.

## Expected Output

Terminal:
```
Simulation active. Non-volatile PIN configuration console online.
```

## Expected Canvas Behavior
* Startup: LCD reads `Code: `.
* Enter `1234`: LCD reads `Access Granted`, Relay clicks ON (GP10 HIGH) for 3 seconds.
* Press `*` (Change Mode): LCD reads `New PIN: `.
* Enter `5555`: LCD reads `PIN Updated!`.
* Restart Simulation and enter `5555`: Door unlocks correctly, indicating non-volatile save success.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `EEPROM.write(10 + i, ...)` | Writes the new user-defined PIN code characters to non-volatile flash addresses starting at index 10. |

## Hardware & Safety Concept: Secure Keypad Interfaces
Adjustable passcode systems must store configurations in non-volatile memories so user changes are not lost when power is turned off. To prevent security vulnerabilities, systems should require entering the **Old PIN** first before allowing the user to configure a new PIN code.

## Try This! (Challenges)
1. **Security Lockout Latch**: Lock inputs for 5 minutes if a wrong PIN is typed 3 times.
2. **Dynamic Backlight**: Turn OFF the LCD backlight automatically if no keys are pressed for 15 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Key presses are doubled | Bounce timing | Ensure that `delay(250)` is executed immediately after key press detection to prevent repeated triggers. |

## Mode Notes
This multi-device I2C and multiplexed project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [106 - Pico Shake Alarm](../intermediate/106-pico-shake-alarm.md)
- [124 - Pico Smart Lock](124-pico-smart-lock.md)
- [127 - Pico Safe Vault](127-pico-safe-vault.md)
