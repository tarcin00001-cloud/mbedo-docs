# 66 - Pico Servo LCD

Control a servo motor's position using a potentiometer and display the live angle on a 16x2 LCD screen.

## Goal
Learn how to read an analog sensor, control a servo motor, and update an I2C text display simultaneously.

## What You Will Build
A digital position pointer console:
- **Potentiometer (GP26)**: Reads 0 to 4095.
- **Servo Motor (GP10)**: Sweeps between 0° and 180°.
- **16x2 I2C LCD (GP4, GP5)**: Displays the current angle (e.g. "Angle: 90 deg") in real time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Potentiometer | Pin 1 (VCC) | 3.3V | Power supply |
| Potentiometer | Pin 2 (Wiper) | GP26 | Analog input |
| Potentiometer | Pin 3 (GND) | GND | Ground return |
| Servo Motor | Signal | GP10 | Servo control |
| Servo Motor | Power | VBUS (5V) | Power supply |
| Servo Motor | Ground | GND | Ground return |
| I2C LCD | VCC | 5V | Display power |
| I2C LCD | SDA | GP4 | I2C Data |
| I2C LCD | SCL | GP5 | I2C Clock |
| I2C LCD | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <Servo.h>
#include <LiquidCrystal_I2C.h>

const int POT_PIN   = 26;
const int SERVO_PIN = 10;

Servo myServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

int lastAngle = -1;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  myServo.attach(SERVO_PIN);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Servo Monitor");
}

void loop() {
  int rawValue = analogRead(POT_PIN);
  int angle = rawValue * 180 / 4095;

  myServo.write(angle);

  // Only redraw LCD if the angle changes (prevents display flickering)
  if (angle != lastAngle) {
    lcd.setCursor(0, 1);
    lcd.print("Angle: ");
    lcd.print(angle);
    lcd.print(" deg   "); // Extra spaces clear old characters
    lastAngle = angle;
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Potentiometer**, **Servo**, and **I2C LCD** onto the canvas.
2. Connect Potentiometer to **GP26**, Servo to **GP10**, and LCD to **GP4/GP5** (SDA/SCL).
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the potentiometer and observe the servo rotate and the LCD update.

## Expected Output

Terminal:
```
Simulation active. LCD rendering servo angle.
```

## Expected Canvas Behavior
| Potentiometer | Servo Position | LCD Row 1 Print |
| --- | --- | --- |
| Far Left | 0° | `Angle: 0 deg` |
| Center | 90° | `Angle: 90 deg` |
| Far Right | 180° | `Angle: 180 deg` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `angle != lastAngle` | Latching condition that limits I2C transmissions to only when data updates. |
| `lcd.print(" deg   ")` | Appends blank spaces to overwrite trailing digits (e.g. going from 180 to 90 leaving the old '0' on screen). |

## Hardware & Safety Concept: Over-Writing Display Buffers
When writing to character displays, printing text directly over existing characters will not erase them if the new text is shorter. For example, printing "100" followed by "95" results in "950" on screen. To prevent this, software developers append trailing spaces to fill character fields, or clear target regions before printing.

## Try This! (Challenges)
1. **Visual Progress Bar**: Draw a simple loading bar on the LCD (e.g., printing `[` and `]` with `=` symbols representing the angle range).
2. **Reverse Speed**: Flash a warning LED on GP15 if the servo reaches its limit boundaries.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD shows garbled digits | Static write loops | Ensure you clear old characters with trailing spaces or call `lcd.clear()` on changes. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [46 - Pico Pot Servo](../../beginner/46-pico-pot-servo.md)
- [58 - Pico LCD Print](58-pico-lcd-print.md)
