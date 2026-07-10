# 97 - Pico Joystick Dual Servo

Build a camera pan-and-tilt bracket controller using a dual-axis analog joystick and two servos.

## Goal
Learn how to read two independent analog coordinates from a joystick and map them to control two separate servo motor angles.

## What You Will Build
A Pan/Tilt joystick controller:
- **Joystick X-Axis (GP26)**: Controls the Pan Servo (GP10) sweep from 0° to 180°.
- **Joystick Y-Axis (GP27)**: Controls the Tilt Servo (GP11) sweep from 0° to 180°.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Analog Joystick Module | `joystick` | Yes | Yes |
| Servo Motor 1 (Pan) | `servo` | Yes | Yes |
| Servo Motor 2 (Tilt) | `servo` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Joystick | VCC | 3.3V | Power supply |
| Joystick | VRX (X-Axis) | GP26 | Pan analog input |
| Joystick | VRY (Y-Axis) | GP27 | Tilt analog input |
| Joystick | GND | GND | Ground reference |
| Pan Servo | Signal | GP10 | Pan servo control |
| Pan Servo | Power | 5V | Power supply |
| Pan Servo | Ground | GND | Ground return |
| Tilt Servo | Signal | GP11 | Tilt servo control |
| Tilt Servo | Power | 5V | Power supply |
| Tilt Servo | Ground | GND | Ground return |

## Code
```cpp
#include <Servo.h>

const int VRX_PIN = 26;
const int VRY_PIN = 27;
const int PAN_PIN = 10;
const int TLT_PIN = 11;

Servo panServo;
Servo tiltServo;

void setup() {
  pinMode(VRX_PIN, INPUT);
  pinMode(VRY_PIN, INPUT);
  
  panServo.attach(PAN_PIN);
  tiltServo.attach(TLT_PIN);
}

void loop() {
  int xVal = analogRead(VRX_PIN); // Reads 0 to 4095
  int yVal = analogRead(VRY_PIN); // Reads 0 to 4095

  // Map 12-bit analog coordinates to Servo Angles (0 to 180)
  int panAngle  = xVal * 180 / 4095;
  int tiltAngle = yVal * 180 / 4095;

  panServo.write(panAngle);
  tiltServo.write(tiltAngle);

  delay(40); // Responsive control update rate
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Joystick**, and **two Servo Motors** onto the canvas.
2. Connect Joystick: **VRX** to **GP26**, **VRY** to **GP27**, **VCC** to **3.3V**, **GND** to **GND**.
3. Connect Pan Servo to **GP10**, Tilt Servo to **GP11**, power lines to **5V**, ground to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Adjust the joystick handles on canvas and watch the two servos turn.

## Expected Output

Terminal:
```
Simulation active. Pan/Tilt joystick controller active.
```

## Expected Canvas Behavior
| Joystick Position | GP26/GP27 Read | Pan Servo (GP10) | Tilt Servo (GP11) |
| --- | --- | --- | --- |
| Center (Idle) | ~2048 / ~2048 | 90° | 90° |
| Left / Center | 0 / 2048 | 0° | 90° |
| Center / Up | 2048 / 0 | 90° | 0° |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `panServo.write(panAngle)` | Sends the mapped angle command based on the joystick X-axis position to the horizontal pan motor. |

## Hardware & Safety Concept: Current Draw Limitations
Driving two servo motors simultaneously under mechanical loads can draw high currents (exceeding 1.5A). Powering both directly from the Pico's 3.3V or 5V VBUS pins without external help can cause voltage drops that reset the Pico. Always connect the servo power lines (red wires) to a dedicated external 5V battery pack, keeping only the GND lines shared with the Pico's ground return.

## Try This! (Challenges)
1. **Incremental Sweeper**: Modify the code so that moving the joystick left or right increments or decrements the current servo angle step-by-step, holding its position when the joystick returns to center.
2. **Alert indicator**: Light a red LED on GP15 if both servos are steered to their limit boundaries.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servos turn in wrong directions | Axis pins swapped | Swap the VRX and VRY pin connections, or swap the servo attach variables in your code. |

## Mode Notes
This multi-device analog motor control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [47 - Pico Joystick X-Axis](../../beginner/47-pico-joystick-x.md)
- [48 - Pico Joystick Y-Axis](../../beginner/48-pico-joystick-y.md)
- [52 - Pico Dual Servo](../intermediate/52-pico-dual-servo.md)
