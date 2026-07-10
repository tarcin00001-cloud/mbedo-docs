# 51 - Pico Servo Sweep

Perform a continuous angular sweep of a servo motor arm.

## Goal
Learn how to program dynamic position changes on servo motor shafts without blocking inputs using timing delays.

## What You Will Build
A standard scanning radar sweep:
- **Servo Motor (GP10)**: Sweeps back and forth continuously between 0 degrees and 180 degrees, simulating a scanning sensor mount.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Servo Motor | Signal (Orange) | GP10 | PWM control |
| Servo Motor | Power (Red) | VBUS (5V) | Power supply |
| Servo Motor | Ground (Brown) | GND | Ground return |

## Code
```cpp
#include <Servo.h>

const int SERVO_PIN = 10;
Servo myServo;

int angle = 0;
int stepDir = 1; // 1 = forward, -1 = backward

void setup() {
  myServo.attach(SERVO_PIN);
  myServo.write(angle);
}

void loop() {
  // Update angle step-by-step to avoid nested loops
  angle = angle + (stepDir * 5); // Increment by 5 degrees

  if (angle >= 180) {
    angle = 180;
    stepDir = -1; // Reverse direction
  }
  if (angle <= 0) {
    angle = 0;
    stepDir = 1;  // Forward direction
  }

  myServo.write(angle);
  delay(50); // Controls sweep speed
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Servo Motor** onto the canvas.
2. Connect Servo: **Signal** to **GP10**, **Power** to **5V**, **Ground** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Observe the servo arm sweeping back and forth.

## Expected Output

Terminal:
```
Simulation active. Sweeping Servo on GP10.
```

## Expected Canvas Behavior
* 0 to 1.8 seconds: Servo moves from 0° to 180°.
* 1.8 to 3.6 seconds: Servo returns from 180° to 0°.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `angle = angle + (stepDir * 5)` | Increments or decrements the current angle variable depending on sweep direction, avoiding blocking nested loops. |

## Hardware & Safety Concept: Gears and Mechanical Limits
Hobby servo motors contain physical plastic or metal gears. Forcing a servo to rotate beyond its mechanical stops (usually just below 0° or slightly above 180°) will strip the gear teeth or stall the internal motor, causing it to overheat and burn out. Always define safety boundaries in software (like limiting range strictly to `0` and `180`).

## Try This! (Challenges)
1. **Speed Sweep**: Modify the delay to 15 ms to make the sweep cycle happen much faster.
2. **Scan Pause**: Program the servo to pause for 1 second every time it reaches 0° and 180°.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo judders at limits | Mechanical limits hit | Restrict the angle range in code to `10` and `170` to verify if the limits are physical. |

## Mode Notes
This basic servo control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [46 - Pico Pot Servo](../../beginner/46-pico-pot-servo.md)
- [52 - Pico Dual Servo](52-pico-dual-servo.md)
