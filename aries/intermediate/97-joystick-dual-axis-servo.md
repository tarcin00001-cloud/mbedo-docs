# 97 - Joystick Dual Axis Servo (Pan & Tilt)

Control the movement of a two-axis pan-and-tilt servo mechanism using an analog joystick on the VEGA ARIES v3 board.

## Goal
Learn how to read dual-channel analog signals from a joystick (X and Y axes), map these values to angular positions (0° to 180°), and drive two servo motors simultaneously using a loop-free C++ structure.

## What You Will Build
A manual control system where moving the joystick left-to-right controls the pan servo (horizontal movement), and moving it up-to-down controls the tilt servo (vertical movement).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Analog Joystick Module | `joystick` | Yes | Yes |
| Servo Motor (Pan) | `servo` | Yes | Yes |
| Servo Motor (Tilt) | `servo` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Joystick | +5V | 5V | Red | Joystick power supply |
| Joystick | GND | GND | Black | Ground reference |
| Joystick | VRx | ADC0 (GP26) | White | X-axis analog output |
| Joystick | VRy | ADC1 (GP27) | Yellow | Y-axis analog output |
| Pan Servo | Signal | GPIO 13 | Orange | Pan PWM control line |
| Pan Servo | VCC | 5V | Red | Servo power supply |
| Pan Servo | GND | GND | Black | Ground reference |
| Tilt Servo | Signal | GPIO 12 | Blue | Tilt PWM control line |
| Tilt Servo | VCC | 5V | Red | Servo power supply |
| Tilt Servo | GND | GND | Black | Ground reference |

> **Wiring tip:** Since you are powering two servos and a joystick, use breadboard power rails to distribute the 5V and GND lines. Ensure both servos share a common ground with the ARIES board.

## Code
```cpp
#include <Servo.h>

Servo servoX;
Servo servoY;

const int JOY_X_PIN = 26; // ADC0 is GP26
const int JOY_Y_PIN = 27; // ADC1 is GP27

void setup() {
  servoX.attach(13); // Attach Pan Servo to GPIO 13
  servoY.attach(12); // Attach Tilt Servo to GPIO 12
}

void loop() {
  int xVal = analogRead(JOY_X_PIN);
  int yVal = analogRead(JOY_Y_PIN);

  // Map 12-bit ADC reading (0 to 4095) to servo angles (0 to 180 degrees)
  int angleX = xVal * 180 / 4095;
  int angleY = yVal * 180 / 4095;

  // Command servos to target positions
  servoX.write(angleX);
  servoY.write(angleY);

  delay(50); // Small interval for physical transition
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **Analog Joystick Module**, and two **Servo Motors** onto the canvas.
2. Wire the Joystick: **+5V** to **5V**, **GND** to **GND**, **VRx** to **ADC0 (GP26)**, and **VRy** to **ADC1 (GP27)**.
3. Wire the Pan Servo: **Signal** to **GPIO 13**, **VCC** to **5V**, and **GND** to **GND**.
4. Wire the Tilt Servo: **Signal** to **GPIO 12**, **VCC** to **5V**, and **GND** to **GND**.
5. Paste the code into the editor.
6. Select **Interpreted Mode** in the simulation dropdown.
7. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Servo X attached to GPIO 13.
Servo Y attached to GPIO 12.
```

## Expected Canvas Behavior
* Moving the simulator's joystick dial along the X-axis rotates the first servo (Pan).
* Moving the joystick dial along the Y-axis rotates the second servo (Tilt) correspondingly.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `servoX.attach(13)` | Sets up the first servo object on physical pin GPIO 13. |
| `servoY.attach(12)` | Sets up the second servo object on physical pin GPIO 12. |
| `analogRead(JOY_X_PIN)` | Reads current horizontal joystick displacement. |
| `xVal * 180 / 4095` | Linearly scales 12-bit ADC range to 180 degrees. |
| `servoX.write(angleX)` | Updates the PWM pulse widths to shift servos to updated angles. |

## Hardware & Safety Concept
* **Dual Servo Current Demands**: Servos draw high currents (up to 500mA–1A each when stalling). Attempting to power multiple servos directly from the ARIES 5V pin under load can cause a brownout reset. For hardware assembly, use an external 5V regulator or power supply dedicated to the servos.
* **Deadzone Handling**: Standard analog joysticks do not always return to exact center positions (2048) due to mechanical spring tolerances. In physical projects, you would implement a small software deadzone threshold to prevent servo jitter when idle.

## Try This! (Challenges)
1. **Joystick Inversion**: Invert one of the control axes so that moving the joystick left rotates the servo right, and moving it down tilts the servo up.
2. **Speed Limiter**: Rather than updating servo angles immediately, limit the step size per loop cycle (e.g., maximum 2° change per 50 ms) to make the pan-and-tilt tracking smoother.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servos jitter when the joystick is in the middle | Mechanical joystick noise | Increase the polling delay or implement a deadzone buffer to ignore minor ADC fluctuations. |
| One servo does not rotate | Pin wiring mismatched | Verify that the Pan Servo signal is on GPIO 13 and Tilt Servo signal is on GPIO 12. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [51 - Servo Motor 180 Degree Sweep](51-servo-motor-180-degree-sweep.md)
- [52 - Dual Servo Coordination Sweep](52-dual-servo-coordination-sweep.md)
