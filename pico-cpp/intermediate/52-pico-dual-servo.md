# 52 - Pico Dual Servo

Synchronize two separate servo motors to sweep in opposite directions.

## Goal
Learn how to manage and write to multiple servo instances simultaneously using separate digital output pins.

## What You Will Build
An alternating pan/tilt scanner:
- **Servo 1 (GP10)**: Sweeps forward from 0 to 180 degrees.
- **Servo 2 (GP11)**: Sweeps backward from 180 to 0 degrees in perfect sync.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Servo Motor 1 | `servo` | Yes | Yes |
| Servo Motor 2 | `servo` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Servo 1 | Signal | GP10 | Servo 1 control |
| Servo 1 | Power | VBUS (5V) | Power supply |
| Servo 1 | Ground | GND | Ground return |
| Servo 2 | Signal | GP11 | Servo 2 control |
| Servo 2 | Power | VBUS (5V) | Power supply |
| Servo 2 | Ground | GND | Ground return |

## Code
```cpp
#include <Servo.h>

const int SERVO1_PIN = 10;
const int SERVO2_PIN = 11;

Servo servo1;
Servo servo2;

int angle = 0;
int stepDir = 1;

void setup() {
  servo1.attach(SERVO1_PIN);
  servo2.attach(SERVO2_PIN);
  
  servo1.write(angle);
  servo2.write(180 - angle);
}

void loop() {
  angle = angle + (stepDir * 5);

  if (angle >= 180) {
    angle = 180;
    stepDir = -1;
  }
  if (angle <= 0) {
    angle = 0;
    stepDir = 1;
  }

  servo1.write(angle);
  servo2.write(180 - angle); // Opposite sweep calculation
  
  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **two Servo Motors** onto the canvas.
2. Connect Servo 1: **Signal** to **GP10**, **Power** to **5V**, **Ground** to **GND**.
3. Connect Servo 2: **Signal** to **GP11**, **Power** to **5V**, **Ground** to **GND**.
4. Paste code, select interpreted mode, and click **Run**.
5. Observe the two servo arms rotating in opposite directions in unison.

## Expected Output

Terminal:
```
Simulation active. Driving dual servos on GP10 and GP11.
```

## Expected Canvas Behavior
| Time | Servo 1 (GP10) | Servo 2 (GP11) |
| --- | --- | --- |
| 0 s | 0° | 180° |
| 0.9 s | 90° | 90° |
| 1.8 s | 180° | 0° |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `servo2.write(180 - angle)` | Inverts the sweep position for the second servo, driving it backward relative to the first. |

## Hardware & Safety Concept: Peak Current Handling
When multiple servos start rotating at the same time, their combined startup current (inrush current) can cause the supply voltage to dip. On real hardware, it is standard practice to connect a large **decoupling capacitor** (e.g., 470uF or 1000uF electrolytic capacitor) across the 5V power rails near the servos. This capacitor stores energy to supply the initial current surge, keeping the microcontroller voltage stable.

## Try This! (Challenges)
1. **Perpendicular Scan**: Modify the second servo calculation so it moves through a range of 45° to 135°.
2. **Speed Mismatch**: Update the step variables so that one servo moves twice as fast as the other.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Microcontroller resets when sweep starts | Voltage dip | Power the servos from an external 5V supply rather than the Pico's 3.3V pin. |

## Mode Notes
This dual servo control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [51 - Pico Servo Sweep](51-pico-servo-sweep.md)
- [97 - Pico Joystick Dual Servo](../../intermediate/97-pico-joystick-servo.md)
