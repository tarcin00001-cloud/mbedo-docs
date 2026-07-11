# 52 - Dual Servo Coordination Sweep

Coordinate two servo motors to sweep in opposite directions simultaneously using a loop-free C++ state machine.

## Goal
Learn how to control and synchronize multiple servo motors on the VEGA ARIES v3 board using the Servo library, ensuring coordinated movements without using loops inside C++ code.

## What You Will Build
A dual-servo coordinated system where Servo 1 sweeps from 0° to 180° while Servo 2 sweeps from 180° to 0° simultaneously, crossing paths exactly at 90°.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Servo Motor (x2) | `servo` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Servo Motor 1 | Signal | GPIO 13 | Yellow | Control line for Servo 1 |
| Servo Motor 1 | VCC | 5V | Red | Power line |
| Servo Motor 1 | GND | GND | Black | Ground |
| Servo Motor 2 | Signal | GPIO 12 | Orange | Control line for Servo 2 |
| Servo Motor 2 | VCC | 5V | Red | Power line |
| Servo Motor 2 | GND | GND | Black | Ground |

> **Wiring tip:** Share the 5V power rails and ground lines on a breadboard. Since two servo motors are active, verify the board is connected to a reliable USB power supply.

## Code
```cpp
#include <Servo.h>

Servo servo1;
Servo servo2;

int angle1 = 0;
int increment = 5;

void setup() {
  servo1.attach(13); // Servo 1 on GPIO 13
  servo2.attach(12); // Servo 2 on GPIO 12
}

void loop() {
  servo1.write(angle1);
  servo2.write(180 - angle1); // Drives Servo 2 in the opposite direction

  angle1 += increment;

  // Reverse direction at the sweep endpoints
  if (angle1 >= 180 || angle1 <= 0) {
    increment = -increment;
  }

  delay(50); // Pause for motor adjustment
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board and two **Servo Motor** components onto the canvas.
2. Connect Servo 1: **Signal** to **GPIO 13**, **VCC** to **5V**, **GND** to **GND**.
3. Connect Servo 2: **Signal** to **GPIO 12**, **VCC** to **5V**, **GND** to **GND**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Servo 1 attached to GPIO 13.
Servo 2 attached to GPIO 12.
```

## Expected Canvas Behavior
* Servo 1 sweeps from 0° to 180° while Servo 2 sweeps from 180° to 0° simultaneously. They cross each other exactly at 90°.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `servo1.attach(13)` | Attaches the first Servo object to GPIO 13. |
| `servo2.attach(12)` | Attaches the second Servo object to GPIO 12. |
| `servo1.write(angle1)` | Directs Servo 1 to rotate to `angle1`. |
| `servo2.write(180 - angle1)` | Directs Servo 2 to rotate to the complementary angle of `angle1`. |
| `angle1 += increment` | Increments the state angle by 5 degrees per loop execution. |

## Hardware & Safety Concept
* **Coordinated Motion Control**: In robotics (such as robotic arms or walking gaits), multiple joints must move in synchronization. Mathematical coordination (like `180 - angle`) ensures proportional movements.
* **Peak Current Draw**: Starting two motors at the exact same instant multiplies the peak surge current. When building physical systems, power high-current actuators from a separate external power rail, leaving the ARIES board to supply the low-current control signals only.

## Try This! (Challenges)
1. **Parallel Sweeping**: Modify the code so both servos rotate in the same direction at the same time (parallel movement).
2. **Speed Ratio**: Adjust the logic so that Servo 2 sweeps at half the speed of Servo 1 (e.g. Servo 2 takes twice as long to complete a full sweep).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Only one servo moves | Pin mismatch or wiring error | Verify Servo 2 is wired specifically to GPIO 12, and check its VCC/GND lines. |
| Servos shake instead of sweeping | Underpowered USB port | Provide external 5V to the power rail or use a stronger USB port/wall adapter. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [51 - Servo Motor 180 Degree Sweep](51-servo-motor-180-degree-sweep.md)
- [53 - DC Motor Start/Stop (via L298N)](53-dc-motor-start-stop.md)
