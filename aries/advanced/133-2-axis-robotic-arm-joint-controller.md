# 133 - 2-Axis Robotic Arm Joint Controller

Control a two-axis (Base and Shoulder) robotic arm joint assembly using two rotary potentiometers and two hobby servo motors.

## Goal
Learn how to sample multiple analog input channels independently, drive multiple servo motors simultaneously, and manage coordinate mapping configurations.

## What You Will Build
A dual-axis robotic arm controller. You will use two potentiometers (wired to `ADC0`/GP26 and `ADC1`/GP27) to control the Base and Shoulder joints of a robotic arm (servos wired to `GPIO 13` and `GPIO 12`). Turning each potentiometer independently adjusts the angle of its respective servo joint, with real-time feedback logged to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| 2x Potentiometers | `potentiometer` | Yes | Yes |
| 2x Hobby Servo Motors | `servo` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer 1 (Base) | VCC / GND | 3V3 / GND | Red / Black | Base input power |
| Potentiometer 1 (Base) | Output / Wiper | ADC0 (GP26) | Yellow | Base position input |
| Potentiometer 2 (Shoulder) | VCC / GND | 3V3 / GND | Red / Black | Shoulder input power |
| Potentiometer 2 (Shoulder) | Output / Wiper | ADC1 (GP27) | Green | Shoulder position input |
| Servo 1 (Base Servo) | VCC / GND | 5V / GND | Red / Black | Base motor power |
| Servo 1 (Base Servo) | Signal | GPIO 13 | Yellow | Base motor control |
| Servo 2 (Shoulder Servo) | VCC / GND | 5V / GND | Red / Black | Shoulder motor power |
| Servo 2 (Shoulder Servo) | Signal | GPIO 12 | Green | Shoulder motor control |

> **Wiring tip:** Share the 3.3V and GND connections for the two potentiometers using the breadboard's power rails. Similarly, share the 5V and GND lines for the two servo motors. Make sure ARIES GND is connected to the shared ground.

## Code
```cpp
// 2-Axis Robotic Arm Joint Controller - VEGA ARIES v3
#include <Servo.h>

const int POT_BASE = GP26;      // Base Potentiometer on ADC0
const int POT_SHOULDER = GP27;  // Shoulder Potentiometer on ADC1

const int SERVO_BASE = 13;      // Base Servo on GPIO 13
const int SERVO_SHOULDER = 12;  // Shoulder Servo on GPIO 12

Servo baseServo;
Servo shoulderServo;

void setup() {
  // Initialize Serial Monitor
  Serial.begin(9600);

  // Attach servo motors to their respective pins
  baseServo.attach(SERVO_BASE);
  shoulderServo.attach(SERVO_SHOULDER);

  Serial.println("2-Axis Robotic Arm Joint Controller online.");
}

void loop() {
  // Read analog inputs from both potentiometers (0 - 4095)
  int valBase = analogRead(POT_BASE);
  int valShoulder = analogRead(POT_SHOULDER);

  // Map values to standard servo angle ranges (0 - 180 degrees)
  int angleBase = map(valBase, 0, 4095, 0, 180);
  int angleShoulder = map(valShoulder, 0, 4095, 0, 180);

  // Command servos to target positions
  baseServo.write(angleBase);
  shoulderServo.write(angleShoulder);

  // Print joint coordinates to the Serial Monitor
  Serial.print("Base -> ADC: ");
  Serial.print(valBase);
  Serial.print(" Angle: ");
  Serial.print(angleBase);
  
  Serial.print(" | Shoulder -> ADC: ");
  Serial.print(valShoulder);
  Serial.print(" Angle: ");
  Serial.println(angleShoulder);

  // Wait 20 ms to allow servo transitions and stabilize reads
  delay(20);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, two **Potentiometers**, and two **Servos** onto the canvas.
2. Wire Potentiometer 1 output to **GP26 (ADC0)**. Wire Potentiometer 2 output to **GP27 (ADC1)**. Connect their power pins to **3V3** and **GND**.
3. Wire Servo 1 (Base): **VCC** to **5V**, **GND** to **GND**, and **Signal** to **GPIO 13**.
4. Wire Servo 2 (Shoulder): **VCC** to **5V**, **GND** to **GND**, and **Signal** to **GPIO 12**.
5. Paste the C++ code into the editor.
6. Select **Interpreted Mode** and click **Run**.
7. Adjust the two virtual potentiometer sliders. Observe the independent rotation of the two virtual servo widgets.

## Expected Output
Serial Monitor:
```
2-Axis Robotic Arm Joint Controller online.
Base -> ADC: 2048 Angle: 90 | Shoulder -> ADC: 1024 Angle: 45
Base -> ADC: 2048 Angle: 90 | Shoulder -> ADC: 3072 Angle: 135
Base -> ADC: 4095 Angle: 180 | Shoulder -> ADC: 3072 Angle: 135
```

## Expected Canvas Behavior
* Adjusting the first potentiometer rotating slider changes only the Servo 1 (Base) angle.
* Adjusting the second potentiometer rotating slider changes only the Servo 2 (Shoulder) angle.
* Both virtual servos move smoothly and independently.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Servo baseServo;` | Declares a Servo object instance to manage base joint positioning pulses. |
| `baseServo.attach(13)` | Assigns base servo motor signal line to GPIO 13. |
| `analogRead(POT_SHOULDER)` | Measures voltage from the shoulder potentiometer on pin GP27 (ADC1). |
| `map(valBase, 0, 4095, 0, 180)` | Translates base joint sensor levels into angles. |
| `shoulderServo.write(...)` | Commands the shoulder joint servo to the target angle. |

## Hardware & Safety Concept
* **Multi-Axis Coordination**: Robotic arm joint coordination relies on fast, independent ADC sampling. In this loop-free configuration, the microcontroller reads and updates both axes sequentially within milliseconds, creating the illusion of parallel execution.
* **Ground Offsets and ADC Accuracy**: Servos draw high pulse currents. If the servo ground lines and potentiometer ground lines share a thin ground wire before reaching the ARIES board, the high current flowing from the servos will cause a momentary voltage shift (ground bounce) on the potentiometer ground pin. This ground bounce introduces electrical noise to the analog reads, causing the servos to jitter. Always wire sensors and actuators directly back to separate board ground pins, or use a solid ground plane.

## Try This! (Challenges)
1. **Mirrored Joint Control**: Modify the code so that Potentiometer 1 controls both servos, but in opposite directions (e.g. Servo 1 moves from 0 to 180 while Servo 2 moves from 180 to 0).
2. **Preset Safety Interlocks**: Program a safety rule where the shoulder servo is not allowed to move below 45 degrees if the base servo is rotated beyond 135 degrees, avoiding mechanical crash hazards.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Moving one potentiometer causes both servos to wiggle | Crosstalk on high-impedance ADC inputs | Ensure the sensors share a solid ground connection. Try placing a small delay (e.g. 5 ms) between `analogRead` calls to allow the ADC sample-and-hold capacitor to discharge. |
| One of the servos is unresponsive | Wrong pin assignment or loose wire | Verify that the base servo is on GPIO 13 and the shoulder servo is on GPIO 12. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [132 - Servo Robotic Arm Base Rotation](132-servo-robotic-arm-base.md)
- [134 - 3-Axis Robotic Arm Claw Controller](134-3-axis-robotic-arm-claw-controller.md)
