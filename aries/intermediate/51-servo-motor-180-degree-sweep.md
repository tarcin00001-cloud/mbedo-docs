# 51 - Servo Motor 180 Degree Sweep

Sweep a servo motor's horn back and forth between 0 and 180 degrees using a loop-free C++ state machine.

## Goal
Learn how to control a servo motor on the VEGA ARIES v3 board using the Servo library, managing the sweep state iteratively in the main loop to conform to interpreted C++ execution rules.

## What You Will Build
An automated sweeping mechanism where a servo motor rotates smoothly from 0° to 180° and back, with the angle updating by 5° every 50 milliseconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Servo Motor | Signal | GPIO 13 | Yellow | Servo PWM control line |
| Servo Motor | VCC | 5V | Red | Servo power supply |
| Servo Motor | GND | GND | Black | Ground reference |

> **Wiring tip:** Servo motors require a stable 5V supply to rotate with full torque. Always verify that VCC connects to the ARIES 5V rail and that grounds are shared.

## Code
```cpp
#include <Servo.h>

Servo myServo;
int angle = 0;
int increment = 5;

void setup() {
  myServo.attach(13); // Attach the servo motor to GPIO 13
}

void loop() {
  myServo.write(angle);
  angle += increment;

  // Reverse sweep direction at mechanical boundaries
  if (angle >= 180 || angle <= 0) {
    increment = -increment;
  }

  delay(50); // Small pause to allow the physical servo to reach the angle
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board and a **Servo Motor** onto the canvas.
2. Connect the Servo: **Signal** to **GPIO 13**, **VCC** to **5V**, and **GND** to **GND**.
3. Paste the C++ code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Servo attached to GPIO 13.
```

## Expected Canvas Behavior
* The Servo Motor's arm rotates back and forth smoothly between 0° and 180° in a continuous, pendulum-like motion.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `#include <Servo.h>` | Imports the standard Servo library for controlling hobby servos. |
| `myServo.attach(13)` | Registers the servo motor on GPIO pin 13. |
| `myServo.write(angle)` | Updates the PWM pulse width to command the servo to the target angle. |
| `angle += increment` | Modifies the target angle in a loop-free state-machine fashion. |
| `increment = -increment` | Reverses the step direction when hitting 0° or 180° boundaries. |

## Hardware & Safety Concept
* **Servo PWM Signals**: Servos are positioned by sending a periodic PWM pulse every 20 ms. The width of this pulse determines the position: a 1.0 ms pulse corresponds to 0°, 1.5 ms to 90°, and 2.0 ms to 180°.
* **Current Isolation**: Servos can draw high stall currents under load, which can cause voltage drops on the microcontroller's power rails. When using multiple servos or heavy mechanical loads, power the servos from an external 5V supply while maintaining a shared ground with the ARIES board.

## Try This! (Challenges)
1. **Speed Tuning**: Change the step size (`increment`) to 2 and decrease the delay to 20 ms for a slower, smoother sweep.
2. **Radar Sweeps**: Modify the logic to sweep rapidly from 0° to 180°, pause at 180° for 1 second, and then sweep back slowly.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo does not move | Incorrect signal pin wired | Verify that the servo's signal wire is connected to GPIO 13, not GPIO 12 or 14. |
| Servo jitters or cuts out | Insufficient operating current | Ensure the servo VCC is connected to the 5V line and the board is powered from a high-quality USB port or external power jack. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [14 - LED PWM Brightness Fade](../beginner/14-led-pwm-brightness-fade.md)
- [52 - Dual Servo Coordination Sweep](52-dual-servo-coordination-sweep.md)
