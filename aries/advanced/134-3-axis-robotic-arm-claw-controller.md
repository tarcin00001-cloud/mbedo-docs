# 134 - 3-Axis Robotic Arm Claw Controller

Build a three-axis robotic arm controller (Base, Shoulder, and Claw) using three analog potentiometers and three independently driven servo motors.

## Goal
Implement multi-channel analog sampling, control three PWM-driven actuators, and develop coordinated articulation logic.

## What You Will Build
A three-joint robotic manipulator control system. The Base, Shoulder, and Claw joints are controlled by three potentiometers wired to `ADC0` (GP26), `ADC1` (GP27), and `ADC2` (GP28). The ARIES board drives three servos (wired to `GPIO 13`, `12`, and `11`) in real-time, mapping joint inputs to precise angular commands.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| 3x Potentiometers | `potentiometer` | Yes | Yes |
| 3x Hobby Servo Motors | `servo` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer 1 (Base) | VCC / GND | 3V3 / GND | Red / Black | Base input power |
| Potentiometer 1 (Base) | Output / Wiper | ADC0 (GP26) | Yellow | Base position input |
| Potentiometer 2 (Shoulder) | VCC / GND | 3V3 / GND | Red / Black | Shoulder input power |
| Potentiometer 2 (Shoulder) | Output / Wiper | ADC1 (GP27) | Green | Shoulder position input |
| Potentiometer 3 (Claw) | VCC / GND | 3V3 / GND | Red / Black | Claw input power |
| Potentiometer 3 (Claw) | Output / Wiper | ADC2 (GP28) | Blue | Claw position input |
| Servo 1 (Base Servo) | VCC / GND | 5V / GND | Red / Black | Base motor power |
| Servo 1 (Base Servo) | Signal | GPIO 13 | Yellow | Base motor control |
| Servo 2 (Shoulder Servo) | VCC / GND | 5V / GND | Red / Black | Shoulder motor power |
| Servo 2 (Shoulder Servo) | Signal | GPIO 12 | Green | Shoulder motor control |
| Servo 3 (Claw Servo) | VCC / GND | 5V / GND | Red / Black | Claw motor power |
| Servo 3 (Claw Servo) | Signal | GPIO 11 | Blue | Claw motor control |

> **Wiring tip:** When working with three analog sensors and three servos, wire cleanliness is critical. Group your sensor wires together and route them away from the servo wires to minimize electrical noise coupling (crosstalk).

## Code
```cpp
// 3-Axis Robotic Arm Claw Controller - VEGA ARIES v3
#include <Servo.h>

const int POT_BASE = GP26;      // Base Potentiometer on ADC0
const int POT_SHOULDER = GP27;  // Shoulder Potentiometer on ADC1
const int POT_CLAW = GP28;      // Claw Potentiometer on ADC2

const int SERVO_BASE = 13;      // Base Servo on GPIO 13
const int SERVO_SHOULDER = 12;  // Shoulder Servo on GPIO 12
const int SERVO_CLAW = 11;      // Claw Servo on GPIO 11

Servo baseServo;
Servo shoulderServo;
Servo clawServo;

void setup() {
  // Initialize Serial Monitor
  Serial.begin(9600);

  // Attach all three servo objects to their pins
  baseServo.attach(SERVO_BASE);
  shoulderServo.attach(SERVO_SHOULDER);
  clawServo.attach(SERVO_CLAW);

  Serial.println("3-Axis Robotic Arm Controller online.");
}

void loop() {
  // Read inputs from the three potentiometers
  int valBase = analogRead(POT_BASE);
  int valShoulder = analogRead(POT_SHOULDER);
  int valClaw = analogRead(POT_CLAW);

  // Map input values (0-4095) to angles (0-180 degrees)
  int angleBase = map(valBase, 0, 4095, 0, 180);
  int angleShoulder = map(valShoulder, 0, 4095, 0, 180);
  int angleClaw = map(valClaw, 0, 4095, 0, 180);

  // Write mapped angles to the corresponding servos
  baseServo.write(angleBase);
  shoulderServo.write(angleShoulder);
  clawServo.write(angleClaw);

  // Log coordinates to the Serial Monitor
  Serial.print("J1: ");
  Serial.print(angleBase);
  Serial.print(" | J2: ");
  Serial.print(angleShoulder);
  Serial.print(" | J3: ");
  Serial.println(angleClaw);

  // Delay to allow servo transit time and limit loop frequency
  delay(20);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, three **Potentiometers**, and three **Servos** onto the canvas.
2. Wire the Potentiometer outputs to ARIES analog pins: **GP26 (ADC0)**, **GP27 (ADC1)**, and **GP28 (ADC2)**. Connect power/ground.
3. Wire the Servos: Servo 1 Signal to **GPIO 13**, Servo 2 Signal to **GPIO 12**, and Servo 3 Signal to **GPIO 11**. Connect VCC to **5V** and GND to **GND**.
4. Paste the C++ code into the editor.
5. Select **Interpreted Mode** and click **Run**.
6. Rotate each virtual potentiometer slider and verify that each corresponds to the correct servo motor.

## Expected Output
Serial Monitor:
```
3-Axis Robotic Arm Controller online.
J1: 90 | J2: 90 | J3: 90
J1: 45 | J2: 90 | J3: 90
J1: 45 | J2: 120 | J3: 15
```

## Expected Canvas Behavior
* Adjusting Potentiometer 1 rotates only the Base Servo (GPIO 13).
* Adjusting Potentiometer 2 rotates only the Shoulder Servo (GPIO 12).
* Adjusting Potentiometer 3 rotates only the Claw Servo (GPIO 11).
* All three servos respond independently to their respective sliders.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Servo clawServo;` | Declares the third Servo instance to control the gripper claw actuator. |
| `clawServo.attach(11)` | Sets up GPIO 11 as the claw servo PWM signal line. |
| `analogRead(POT_CLAW)` | Samples the third analog channel on pin GP28 (ADC2) to read the claw position. |
| `map(valClaw, 0, 4095, 0, 180)` | Translates the claw sensor reading to a target angle. |
| `clawServo.write(angleClaw)` | Commands the claw servo to clamp or release. |

## Hardware & Safety Concept
* **Robotic Gripper/Claw Limits**: Mechanical claws have restricted physical boundaries. Driving a claw servo to a theoretical limit (like 0° or 180°) when the claw is physically closed can stall the motor, leading to overheating, high current draw, and servo damage. It is crucial to determine the physical opening/closing angles of your claw and limit the code map limits accordingly (e.g. `map(valClaw, 0, 4095, 40, 130)`).
* **Power Supply Integrity (Brownouts)**: Driving three servos simultaneously under mechanical load requires significant power (exceeding 2A peak currents). A standard USB port only provides up to 500mA. Running this on physical hardware without an external, high-current 5V supply will trigger instant microcontroller brownout resets when all three joints move together.

## Try This! (Challenges)
1. **Claw Clamp Limiter**: Modify the code to restrict the claw servo mapping bounds to `50` (fully closed) and `130` (fully open) to protect the physical claw mechanism from stalling.
2. **Coordinated Motion (Mirror Joints)**: Set up a tracking mode where turning Potentiometer 1 moves both the Base (J1) and Shoulder (J2) joints synchronously to execute diagonal sweeps.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servos jitter or move on their own | Electrical noise on analog input lines | Ensure all three potentiometer grounds are securely connected to the ARIES board GND. Try adding a 0.1 µF bypass capacitor across each potentiometer output pin and GND. |
| The Claw servo does not move, but the base and shoulder do | Incorrect pin wiring | Verify that the claw servo signal wire is plugged into GPIO 11, and that `POT_CLAW` reads GP28. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [133 - 2-Axis Robotic Arm Joint Controller](133-2-axis-robotic-arm-joint-controller.md)
- [135 - Robotic Arm Position Memory Log](135-robotic-arm-position-memory-log.md)
