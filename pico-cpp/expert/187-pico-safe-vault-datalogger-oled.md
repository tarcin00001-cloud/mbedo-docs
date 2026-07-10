# 187 - Pico Safe Vault Datalogger OLED

Build an advanced safe vault controller that saves configurable access PIN codes in flash (EEPROM), displays entry indicators on an OLED, and streams audit logs over Bluetooth.

## Goal
Learn how to read keypads, store values in non-volatile memory (EEPROM), actuate solenoids, design OLED check menus, and stream real-time access security reports over Bluetooth.

## What You Will Build
A security safe with wireless audit logs and OLED HUD:
- **Relay Module (GP10)**: Connects to the vault door solenoid strike.
- **4x4 Keypad**: Receives passcode entries.
- **EEPROM (Flash)**: Saves the active 4-digit PIN code (default `1234`) at address `10`.
- **SSD1306 OLED (GP4, GP5)**: Displays status menus, credentials, and asterisks for PIN entries.
- **HC-05 Bluetooth Module (GP0, GP1)**: Streams audit log entries.
- **Active Buzzer (GP14)**: Sound chime feedback.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Keypad Rows | R1 - R4 | GP2 / GP3 / GP6 / GP7 | Output scanning rows |
| Keypad Cols | C1 - C4 | GP8 / GP9 / GP20 / GP21 | Input columns (pull-up) |
| Relay Module | IN | GP10 | Electronic lock solenoid |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
| Active Buzzer | VCC (+) | GP14 | Alarm chime pin |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <EEPROM.h>

const int RELAY_PIN = 10;
const int BUZZ_PIN  = 14;

// Keypad pins
const int R1 = 2; const int R2 = 3; const int R3 = 6; const int R4 = 7;
const int C1 = 8; const int C2 = 9; const int C3 = 20; const int C4 = 21;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

char storedPIN[5]  = "1234";
char enteredPIN[5] = "    ";
int pinCount = 0;
bool changeMode = false;

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART

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

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }

  EEPROM.begin(512);

  // Load PIN from EEPROM (address 10-13)
  for (int i = 0; i < 4; i++) {
    char val = EEPROM.read(10 + i);
    if (val >= '0' && val <= '9') {
      storedPIN[i] = val;
    }
  }

  showIdleScreen();
  Serial1.println("Safe Audit Gateway Online.");
}

void loop() {
  char key = scanKeypad();

  if (key != '\0') {
    tone(BUZZ_PIN, 800, 80);
    delay(250); // Key debounce

    if (key == '*') {
      changeMode = true;
      pinCount = 0;
      
      display.clearDisplay();
      display.fillRect(0, 0, 128, 64, SSD1306_WHITE);
      display.setTextColor(SSD1306_BLACK);
      display.setCursor(10, 15);
      display.print("Change PIN Mode");
      display.setCursor(10, 35);
      display.print("New PIN: ");
      display.display();
    } 
    else if (key == '#') {
      pinCount = 0;
      showIdleScreen();
    } 
    else {
      enteredPIN[pinCount] = key;
      pinCount = pinCount + 1;
      
      display.clearDisplay();
      if (changeMode) {
        display.fillRect(0, 0, 128, 64, SSD1306_WHITE);
        display.setTextColor(SSD1306_BLACK);
        display.setCursor(10, 15);
        display.print("Change PIN Mode");
        display.setCursor(10, 35);
        display.print("New PIN: ");
        display.setCursor(75, 35);
      } else {
        display.setTextColor(SSD1306_WHITE);
        display.setCursor(15, 15);
        display.print("SAFE VAULT SECURE");
        display.setCursor(15, 35);
        display.print("Code: ");
        display.setCursor(55, 35);
      }
      
      if (pinCount >= 1) display.print("*");
      if (pinCount >= 2) display.print("*");
      if (pinCount >= 3) display.print("*");
      if (pinCount >= 4) display.print("*");
      
      if (!changeMode) {
        display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
      }
      display.display();

      if (pinCount >= 4) {
        display.clearDisplay();

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

          display.fillRect(0, 0, 128, 64, SSD1306_WHITE);
          display.setTextColor(SSD1306_BLACK);
          display.setCursor(15, 20);
          display.print("PIN Updated!");
          display.display();
          
          Serial1.println("EVENT:PIN_CHANGED");
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
            display.fillRect(0, 0, 128, 64, SSD1306_WHITE);
            display.setTextColor(SSD1306_BLACK);
            display.setCursor(15, 20);
            display.print("ACCESS GRANTED");
            display.display();
            
            digitalWrite(RELAY_PIN, HIGH); // Open door strike
            Serial1.println("EVENT:ACCESS_GRANTED");
            tone(BUZZ_PIN, 1000, 500);
            delay(3000);
            digitalWrite(RELAY_PIN, LOW); // Lock door strike
          } else {
            display.setTextColor(SSD1306_WHITE);
            display.setCursor(15, 20);
            display.print("ACCESS DENIED!");
            display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
            display.display();
            
            Serial1.println("EVENT:ACCESS_DENIED_WRONG_PIN");
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
  
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(15, 15);
  display.print("SAFE VAULT SECURE");
  display.setCursor(15, 35);
  display.print("Code: ");
  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.display();
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
1. Drag the **Raspberry Pi Pico**, **4x4 Keypad**, **Relay**, **SSD1306 OLED**, **HC-05**, **Active Buzzer**, and **I2C LCD** onto the canvas.
2. Connect sensors, relays, screen, and Bluetooth.
3. Paste code, select the interpreted mode, and click **Run**.
4. Enter passcodes, check if the audit log prints over Bluetooth, then test changing the PIN.

## Expected Output

Terminal (Bluetooth):
```
Safe Audit Gateway Online.
EVENT:ACCESS_GRANTED
EVENT:PIN_CHANGED
EVENT:ACCESS_DENIED_WRONG_PIN
```

## Expected Canvas Behavior
* Normal state: OLED reads `Code: `.
* Enter `1234`: OLED reads `ACCESS GRANTED` on a white background, Relay clicks ON (GP10 HIGH) for 3 seconds. Bluetooth prints `EVENT:ACCESS_GRANTED`.
* Change PIN (`*` followed by `5555`): OLED reads `PIN Updated!`. Bluetooth prints `EVENT:PIN_CHANGED`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Serial1.println("EVENT:ACCESS_GRANTED")` | Sends a real-time event log packet over the Bluetooth UART channel for security auditing. |

## Hardware & Safety Concept: Secure Keypad Interfaces
Adjustable passcode systems must store configurations in non-volatile memories so user changes are not lost when power is turned off. To prevent security vulnerabilities, systems should require entering the **Old PIN** first before allowing the user to configure a new PIN code.

## Try This! (Challenges)
1. **Latching Tamper Lock**: Disable keypad scans for 5 minutes if an incorrect PIN is entered 3 times.
2. **Door Open Timeout Alert**: Sound a warning beep if the door strike remains active for too long.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Double key registration | Key bounce issues | Verify that the debounce delay (`delay(250)`) runs after key detection to let contact contacts settle. |

## Mode Notes
This multi-device I2C and multiplexed project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [124 - Pico Smart Lock](../advanced/124-pico-smart-lock.md)
- [127 - Pico Safe Vault](../advanced/127-pico-safe-vault.md)
- [157 - Pico Safe Vault EEPROM](../advanced/157-pico-safe-vault-eeprom.md)
- [175 - Pico Safe Vault Datalogger](175-pico-safe-vault-datalogger.md)
