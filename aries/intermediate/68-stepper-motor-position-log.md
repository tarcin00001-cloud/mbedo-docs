# 68 - Stepper Motor Position Log (LCD display)

Track and log the cumulative step count of a stepper motor on an I2C LCD screen using the VEGA ARIES v3 board.

## Goal
Learn how to track step states in a C++ state machine, drive a stepper motor sequentially, and log positional data on an I2C display without using loops inside C++ code.

## What You Will Build
A position tracking system that drives a stepper motor forward step-by-step and displays the running step count on a 16x2 character LCD screen in real time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| ULN2003 Driver Board | `uln2003` | Yes | Yes |
| 28BYJ-48 Stepper Motor | `stepper` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| ULN2003 Driver | IN1 | GPIO 14 | Blue | Phase 1 control line |
| ULN2003 Driver | IN2 | GPIO 15 | Yellow | Phase 2 control line |
| ULN2003 Driver | IN3 | GPIO 13 | Orange | Phase 3 control line |
| ULN2003 Driver | IN4 | GPIO 12 | Green | Phase 4 control line |
| ULN2003 Driver | VCC | 5V | Red | Driver power supply |
| ULN2003 Driver | GND | GND | Black | Ground reference |
| I2C LCD | VCC | 5V | Red | LCD power supply |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | SDA0 (GP17) | Blue | I2C Data line |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** Share the 5V power rails and GND lines on a breadboard. Keep all stepper phase wires organized to prevent ordering errors.

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int IN1 = 14;
const int IN2 = 15;
const int IN3 = 13;
const int IN4 = 12;

LiquidCrystal_I2C lcd(0x27, 16, 2);
int stepIndex = 0;
int stepCount = 0;

void setup() {
  Wire.begin(); // Initialize default I2C0 interface

  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Stepper Log");
}

void loop() {
  // Stepper Wave Drive Sequence (Single-Phase Stepping)
  if (stepIndex == 0) {
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, LOW);
  } else if (stepIndex == 1) {
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, HIGH);
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, LOW);
  } else if (stepIndex == 2) {
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, HIGH);
    digitalWrite(IN4, LOW);
  } else if (stepIndex == 3) {
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, HIGH);
  }

  stepIndex = (stepIndex + 1) % 4; // Advance stepper phase index
  stepCount = stepCount + 1;       // Increment cumulative step count

  // Display log count on row 1
  lcd.setCursor(0, 1);
  lcd.print("Steps: ");
  lcd.print(stepCount);
  lcd.print("      "); // Trailing spaces to overwrite old text digits

  delay(50); // Speed controls (50 ms per step)
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **ULN2003 Driver**, **28BYJ-48 Stepper Motor**, and **I2C LCD Display** components onto the canvas.
2. Wire the ULN2003: **IN1-IN4** to **GPIO 14, 15, 13, 12**, **VCC** to **5V**, and **GND** to **GND**. Connect the motor.
3. Wire the LCD: **VCC** to **5V**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Step Logged: 1
Step Logged: 2
```

## Expected Canvas Behavior
* The stepper motor spins clockwise. The LCD displays `Stepper Log` on row 0 and updates the step count on row 1 (e.g. `Steps: 45`).

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(...)` | Initializes the stepper driver output pins. |
| `digitalWrite(IN1, HIGH)` | Drives phase 1 coil. |
| `stepCount = stepCount + 1` | Increments the absolute step count tracking variable. |
| `lcd.print(stepCount)` | Updates the count characters on the LCD display. |
| `delay(50)` | Pause interval that dictates stepping frequency and display update rate. |

## Hardware & Safety Concept
* **Open-Loop Positioning**: Stepper motors operate in an open-loop fashion, meaning they rotate in fixed angular steps and assume the movement occurred without feedback sensors (like encoders). Tracking step count in code provides a software-calculated estimate of angular position (e.g., a 2048-step gear reduction motor has moved 90 degrees after 512 steps).
* **Driver Isolation**: ULN2003 driver arrays prevent high-voltage motor currents from flowing back into the ARIES SoC pins. Always keep the driver board and LCD module ground paths unified.

## Try This! (Challenges)
1. **Degree Logger**: Map the step count to degrees (e.g., if a motor has 2048 steps per full 360° turn, calculate `angle = stepCount * 360 / 2048`). Display `Angle: XX deg` instead of raw steps.
2. **Direction Control**: Add a button on GPIO 16 that toggles the direction. Update the step count to decrement when the motor rotates counter-clockwise.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Display shows steps updating but motor does not turn | Motor power unconnected | Make sure the ULN2003 module is connected to the 5V line and the board is powered. |
| Count increments but values do not clear on screen | Missing trailing spaces | Ensure the LCD print function has trailing spaces (e.g., `"      "`) to clear characters when count wraps. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [56 - Stepper Motor Step Sequence (ULN2003)](56-stepper-motor-step-sequence.md)
- [66 - Servo Position Display](66-servo-position-display.md)
