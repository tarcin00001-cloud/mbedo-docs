# 66 - Servo Position Display

Sweep a servo motor and display its live angle on a 16x2 I2C Character LCD using the VEGA ARIES v3 board.

## Goal
Learn how to control a servo motor on GPIO 13, initialize I2C LCD communication, and write text HUD outputs that update dynamically without using loops inside C++ code.

## What You Will Build
A digital positioning display system where a servo sweeps between 0° and 180° in 10° increments, and its live angle is printed onto a 16x2 LCD screen in real time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Servo Motor | Signal | GPIO 13 | Yellow | Servo control line |
| Servo Motor | VCC | 5V | Red | Servo power supply |
| Servo Motor | GND | GND | Black | Ground reference |
| I2C LCD | VCC | 5V | Red | LCD power supply |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | SDA0 (GP17) | Blue | I2C Data line |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** Share the 5V and GND pins of the ARIES board using a breadboard to power both the servo motor and the I2C LCD.

## Code
```cpp
#include <Wire.h>
#include <Servo.h>
#include <LiquidCrystal_I2C.h>

Servo myServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

int angle = 0;
int increment = 10;

void setup() {
  Wire.begin(); // Initialize default I2C0 interface

  myServo.attach(13); // Servo signal connected to GPIO 13

  lcd.init();
  lcd.backlight();

  lcd.setCursor(0, 0);
  lcd.print("Servo Position");
}

void loop() {
  myServo.write(angle);

  // Update position readout on row 1
  lcd.setCursor(0, 1);
  lcd.print("Angle: ");
  lcd.print(angle);
  lcd.print(" deg   "); // Trailing spaces overwrite old digits (e.g. overwriting 180 with 90)

  angle += increment;

  // Reverse directions at endpoints
  if (angle >= 180 || angle <= 0) {
    increment = -increment;
  }

  delay(200); // Sweep interval delay (allows servo to reach target position)
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **Servo Motor**, and **I2C LCD Display** components onto the canvas.
2. Connect the Servo: **Signal** to **GPIO 13**, **VCC** to **5V**, and **GND** to **GND**.
3. Connect the LCD: **VCC** to **5V**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Servo Sweeping. Current Angle: 0 deg
Servo Sweeping. Current Angle: 10 deg
```

## Expected Canvas Behavior
* The servo sweeps back and forth. The 16x2 LCD shows `Servo Position` on the first row, and updates `Angle: [Current Angle] deg` on the second row.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `myServo.attach(13)` | Registers the servo control on GPIO pin 13. |
| `lcd.init()` | Initializes the I2C LCD controller interface. |
| `myServo.write(angle)` | Moves the servo to the target angle. |
| `lcd.print(" deg   ")` | Prints units; trailing spaces ensure values like 180 clear properly to 90 (preventing "900 deg" artifact). |
| `delay(200)` | Controls sweep speed and gives the servo horn time to transit. |

## Hardware & Safety Concept
* **LCD Screen Ghosting**: Character LCD screens can exhibit "ghosting" or text artifacts if values are printed over old values without clearing them. Printing trailing spaces (e.g. `deg   `) is more efficient than calling `lcd.clear()`, which takes up to 2 ms and causes the screen to flicker.
* **Inductive Load Noise**: Servos draw high spikes of current when starting. This can create voltage dips on the I2C bus lines, causing the LCD controller to crash or display garbage characters. Adding a decoupling capacitor or keeping the wiring separate helps maintain stable bus communications.

## Try This! (Challenges)
1. **Interactive Dial**: Modify the code to read a Potentiometer on `ADC0 (GP26)` and map its output to the servo position. Display the mapped angle on the LCD.
2. **Dual-speed Sweeper**: Sweep from 0° to 180° in 5° steps, but return from 180° to 0° in 20° steps.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Angle value on screen shows trailing digits (e.g. "900 deg" instead of "90 deg") | Missing trailing spaces | Ensure the print statement contains trailing spaces (e.g., `" deg   "`) to overwrite old characters. |
| LCD backlight is on but characters are blank | Mismatched address or low contrast | Adjust the contrast screw on the back of the LCD module, or verify that the address parameter is `0x27`. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [51 - Servo Motor 180 Degree Sweep](51-servo-motor-180-degree-sweep.md)
- [67 - Potentiometer Speed DC Motor (with LCD)](67-potentiometer-speed-dc-motor.md)
