# 127 - Pico Safe Vault

Build a secure vault vault alarm node that sounds a siren on motion/vibration detect, requiring a secret keypad PIN override to silence.

## Goal
Learn how to implement latching alarm security systems using multiple inputs (PIR motion, spring vibration), driving alerts, and processing loop-free keypad PIN validation.

## What You Will Build
A security safe alarm system:
- **PIR Motion Sensor (GP16)**: Detects movement near the safe.
- **Vibration Sensor (GP17)**: Detects physical safe tampering (drilling/bumping).
- **Active Buzzer (GP14)**: Sounds a loud pulsing alarm when a breach is detected.
- **4x4 Keypad (GP2, GP3, GP6, GP7, GP8, GP9, GP20, GP21)**: Silences the alarm when the PIN `5555` is entered.
- **16x2 I2C LCD (GP4, GP5)**: Displays status (e.g. "VAULT BREACHED" or "SAFE SECURED").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |
| Vibration Sensor Module | `button` | Yes (represented by switch button) | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| PIR Sensor | OUT | GP16 | Motion sense |
| Vibration Sensor | OUT | GP17 | Tamper sense |
| Active Buzzer | VCC (+) | GP14 | Alarm pin |
| Keypad Rows | R1 - R4 | GP2 / GP3 / GP6 / GP7 | Output scan rows |
| Keypad Cols | C1 - C4 | GP8 / GP9 / GP20 / GP21 | Input columns (pull-up) |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen |
| All Modules | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int PIR_PIN    = 16;
const int VIB_PIN    = 17;
const int BUZZ_PIN   = 14;

// Keypad pins
const int R1 = 2; const int R2 = 3; const int R3 = 6; const int R4 = 7;
const int C1 = 8; const int C2 = 9; const int C3 = 20; const int C4 = 21;

LiquidCrystal_I2C lcd(0x27, 16, 2);

bool alarmLatched = false;
int pinCount = 0;
char enteredPIN[5] = "    ";
char targetPIN[5]  = "5555";

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(PIR_PIN, INPUT);
  pinMode(VIB_PIN, INPUT);
  pinMode(BUZZ_PIN, OUTPUT);
  digitalWrite(BUZZ_PIN, LOW);

  // Setup keypad pins
  pinMode(R1, OUTPUT); pinMode(R2, OUTPUT); pinMode(R3, OUTPUT); pinMode(R4, OUTPUT);
  pinMode(C1, INPUT_PULLUP); pinMode(C2, INPUT_PULLUP); pinMode(C3, INPUT_PULLUP); pinMode(C4, INPUT_PULLUP);

  lcd.init();
  lcd.backlight();
  resetSafeState();
}

void loop() {
  int motion = digitalRead(PIR_PIN);
  int tamper = digitalRead(VIB_PIN); // Active-LOW spring contact check

  // Trigger alarm on motion or vibration
  if ((motion == HIGH || tamper == LOW) && !alarmLatched) {
    alarmLatched = true;
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("! VAULT BREACH !");
    lcd.setCursor(0, 1);
    lcd.print("Enter PIN: ");
    pinCount = 0;
  }

  // If alarm is active, run siren and read keypad override
  if (alarmLatched) {
    // Pulse siren
    digitalWrite(BUZZ_PIN, HIGH);
    delay(80);
    digitalWrite(BUZZ_PIN, LOW);
    
    // Read keypad key
    char key = scanKeypad();
    if (key != '\0') {
      tone(BUZZ_PIN, 800, 80);
      enteredPIN[pinCount] = key;
      pinCount = pinCount + 1;
      
      lcd.setCursor(11 + pinCount - 1, 1);
      lcd.print("*");
      delay(300); // Key debounce

      if (pinCount >= 4) {
        // Check PIN
        bool match = true;
        if (enteredPIN[0] != targetPIN[0]) { match = false; }
        if (enteredPIN[1] != targetPIN[1]) { match = false; }
        if (enteredPIN[2] != targetPIN[2]) { match = false; }
        if (enteredPIN[3] != targetPIN[3]) { match = false; }

        if (match) {
          lcd.clear();
          lcd.setCursor(0, 0);
          lcd.print("Safe Secured");
          tone(BUZZ_PIN, 1200, 400);
          delay(2000);
          resetSafeState();
        } else {
          lcd.clear();
          lcd.setCursor(0, 0);
          lcd.print("Access Denied!");
          tone(BUZZ_PIN, 150, 1000);
          delay(2000);
          lcd.clear();
          lcd.setCursor(0, 0);
          lcd.print("! VAULT BREACH !");
          lcd.setCursor(0, 1);
          lcd.print("Enter PIN: ");
          pinCount = 0;
        }
      }
    }
  } else {
    digitalWrite(BUZZ_PIN, LOW);
  }

  delay(20);
}

void resetSafeState() {
  alarmLatched = false;
  pinCount = 0;
  enteredPIN[0] = ' '; enteredPIN[1] = ' '; enteredPIN[2] = ' '; enteredPIN[3] = ' ';
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Safe Vault Mode");
  lcd.setCursor(0, 1);
  lcd.print("Monitoring...   ");
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
1. Drag the **Raspberry Pi Pico**, **PIR Sensor**, **Vibration Sensor** (switch button), **4x4 Keypad**, **Active Buzzer**, and **I2C LCD** onto the canvas.
2. Connect PIR OUT to **GP16**, Vibration OUT to **GP17**, and control lines as wired.
3. Paste code, select the interpreted mode, and click **Run**.
4. Simulate safe tampering by closing the vibration switch, then enter the PIN code `5555` on the keypad to silence the siren.

## Expected Output

Terminal:
```
Simulation active. Latching safe alarm monitor active.
```

## Expected Canvas Behavior
* Normal state: LCD reads `Safe Vault Mode` / `Monitoring...`. Silent.
* Vibration triggered: LCD changes to `! VAULT BREACH !` / `Enter PIN: `, buzzer pulses rapidly.
* PIN `5555` entered: LCD reads `Safe Secured`, buzzer turns OFF.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `motion == HIGH \|\| tamper == LOW` | Checks if either movement or physical vibration is detected to trigger the latching security alarm. |

## Hardware & Safety Concept: Tamper-Resistant Safety Logic
Vibration sensors detect physical drilling, hammering, or sawing attempts on secure metal safes. In commercial vaults, these sensors run in a **Normally Closed (NC)** loop. If an intruder cuts the sensor wire, the loop opens (breaks), which triggers the alarm immediately. This is much safer than Open loops, where cut wires render the system useless.

## Try This! (Challenges)
1. **Wrong PIN Alarm Latch**: Lock the keypad inputs out for 5 minutes if an incorrect PIN is typed 3 times.
2. **Auto Dial Alert**: Add a Bluetooth module and send a wireless "INTRUDER BREACH ALERT!" message to a remote terminal.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm won't turn OFF on correct PIN | Debounce register mismatch | Verify that the target PIN matches `5555` exactly and that you are scanning row/cols correctly without wire overlaps. |

## Mode Notes
This multi-device I2C and multiplexed project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [106 - Pico Shake Alarm](../intermediate/106-pico-shake-alarm.md)
- [124 - Pico Smart Lock](124-pico-smart-lock.md)
