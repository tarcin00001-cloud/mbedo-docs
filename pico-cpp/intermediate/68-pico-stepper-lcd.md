# 68 - Pico Stepper LCD

Control the step position of a stepper motor using a potentiometer and display the coordinates on an LCD.

## Goal
Learn how to index absolute positions on stepper motors from analog inputs and update system position metrics on character displays.

## What You Will Build
A digital positioning knob:
- **Potentiometer (GP26)**: Mapped to target steps.
- **Stepper Motor (GP12-15)**: Shaft turns to align with target coordinate steps.
- **16x2 I2C LCD (GP4, GP5)**: Displays target steps (e.g. "Pos: 200 steps").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| Stepper Motor | `stepper` | Yes | Yes |
| ULN2003 Driver Module | `uln2003` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Potentiometer | Pin 1 (VCC) | 3.3V | Power supply |
| Potentiometer | Pin 2 (Wiper) | GP26 | Analog input |
| Potentiometer | Pin 3 (GND) | GND | Ground return |
| ULN2003 | IN1-IN4 | GP12-GP15 | Stepper control |
| ULN2003 | GND | GND | Ground return |
| I2C LCD | VCC | 5V | Display power |
| I2C LCD | SDA | GP4 | I2C Data |
| I2C LCD | SCL | GP5 | I2C Clock |
| I2C LCD | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int POT_PIN = 26;
const int IN1 = 12;
const int IN2 = 13;
const int IN3 = 14;
const int IN4 = 15;

LiquidCrystal_I2C lcd(0x27, 16, 2);

int currentStep = 0;
int lastTarget = -1;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(POT_PIN, INPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Stepper Position");
}

void loop() {
  int rawValue = analogRead(POT_PIN);
  
  // Map 12-bit input to 0-400 steps range
  int targetStep = rawValue * 400 / 4095;

  // Move stepper towards target position step-by-step
  if (currentStep < targetStep) {
    stepForward();
    currentStep = currentStep + 1;
    delay(10);
  }
  else if (currentStep > targetStep) {
    stepBackward();
    currentStep = currentStep - 1;
    delay(10);
  }

  // Update display if target coordinate updates
  if (targetStep != lastTarget) {
    lcd.setCursor(0, 1);
    lcd.print("Pos: ");
    lcd.print(targetStep);
    lcd.print(" steps    ");
    lastTarget = targetStep;
  }

  delay(10);
}

// Single step forward helper logic inside main loop (no custom functions on actual shims, but helper loops fit interpreter)
void stepForward() {
  int phase = currentStep % 4;
  if (phase == 0) { digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  digitalWrite(IN3, LOW);  digitalWrite(IN4, LOW); }
  if (phase == 1) { digitalWrite(IN1, LOW);  digitalWrite(IN2, HIGH); digitalWrite(IN3, LOW);  digitalWrite(IN4, LOW); }
  if (phase == 2) { digitalWrite(IN1, LOW);  digitalWrite(IN2, LOW);  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW); }
  if (phase == 3) { digitalWrite(IN1, LOW);  digitalWrite(IN2, LOW);  digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH); }
}

void stepBackward() {
  int phase = currentStep % 4;
  if (phase == 0) { digitalWrite(IN1, LOW);  digitalWrite(IN2, LOW);  digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH); }
  if (phase == 1) { digitalWrite(IN1, LOW);  digitalWrite(IN2, LOW);  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW); }
  if (phase == 2) { digitalWrite(IN1, LOW);  digitalWrite(IN2, HIGH); digitalWrite(IN3, LOW);  digitalWrite(IN4, LOW); }
  if (phase == 3) { digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  digitalWrite(IN3, LOW);  digitalWrite(IN4, LOW); }
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Potentiometer**, **ULN2003**, **Stepper**, and **I2C LCD** onto the canvas.
2. Connect Potentiometer to **GP26**, ULN2003 to **GP12-15**, and LCD to **GP4/5**.
3. Paste code, select interpreted mode, and click **Run**.
4. Adjust the potentiometer dial to position the stepper shaft and monitor position on the LCD.

## Expected Output

Terminal:
```
Simulation active. LCD tracking stepper steps.
```

## Expected Canvas Behavior
| Potentiometer | Target Steps | Stepper Direction | LCD Row 1 Print |
| --- | --- | --- | --- |
| Far Left | 0 | Moves to 0 | `Pos: 0 steps` |
| Center | 200 | Moves to 200 | `Pos: 200 steps` |
| Far Right | 400 | Moves to 400 | `Pos: 400 steps` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `currentStep < targetStep` | Control loop comparing current step memory against mapped target steps to drive direction. |

## Hardware & Safety Concept: Stepper Positioning Accuracy
Stepper motors provide high precision positioning without feedback sensors because each step command turns the shaft by a fixed mechanical angle (e.g. 5.625° on the 28BYJ-48 gear down). However, if the motor hits a physical barrier or is overloaded, it will "slip" (lose steps). Because there is no feedback loop, the microcontroller has no way of knowing the motor has slipped, resulting in position drift until the system is re-homed (recalibrated).

## Try This! (Challenges)
1. **Home Switch**: Add a limit switch on GP16. If pressed, reset the `currentStep` to 0.
2. **Speed Adjustment**: Modify the stepping delay to 5 ms to make the motor track the target faster.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor moves in wrong direction | Reversed wiring | Swap Phase lines IN1/IN4 and IN2/IN3 in wiring, or invert the indexing phases inside the code. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [57 - Pico Stepper Speed](57-pico-stepper-speed.md)
- [66 - Pico Servo LCD](66-pico-servo-lcd.md)
