# 123 - Robot Speed Control

Vary the speed of a mobile robot by driving the enable pins (ENA and ENB) of an L298N motor driver with PWM signals.

## Goal
Learn how to regulate DC motor speed using Pulse Width Modulation (PWM) on H-bridge enable lines, configure speed scaling parameters, and code speed-shifting profiles.

## What You Will Build
A speed-regulated mobile robot. The ARIES board controls the L298N driver. In addition to direction control pins (IN1-IN4), enable pins (ENA and ENB) are connected to PWM-capable outputs on the ARIES board (GPIO 11 and GPIO 10). The robot steps through different speed levels (Stop, Slow, Medium, Full Speed) every 2 seconds, displaying the speed changes on both the motors and the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| L298N Motor Driver Module | `l298n` | Yes | Yes |
| 2x DC Toy Motors | `dc_motor` | Yes | Yes |
| External 6V-12V Power Source | `power_supply` | Optional | Yes (battery pack) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Module | VCC (Power) | 5V | Red | Driver logic/motor power |
| L298N Module | GND | GND | Black | Shared ground connection |
| L298N Module | ENA (Enable A) | GPIO 11 | Yellow | Left motor speed control (remove jumper) |
| L298N Module | IN1 | GPIO 14 | Orange | Left motor forward control |
| L298N Module | IN2 | GPIO 15 | Brown | Left motor reverse control |
| L298N Module | ENB (Enable B) | GPIO 10 | Yellow | Right motor speed control (remove jumper) |
| L298N Module | IN3 | GPIO 13 | Green | Right motor forward control |
| L298N Module | IN4 | GPIO 12 | Blue | Right motor reverse control |
| Left DC Motor | Pin 1 / Pin 2 | OUT1 / OUT2 | Red / Black | Left motor output terminals |
| Right DC Motor | Pin 1 / Pin 2 | OUT3 / OUT4 | Red / Black | Right motor output terminals |

> **Wiring tip:** To use PWM speed control, you must remove the black plastic jumpers pre-installed on the L298N ENA and ENB pins. Connect these pins directly to the ARIES board's PWM pins (GPIO 11 and 10).

## Code
```cpp
// Robot Speed Control - VEGA ARIES v3
const int IN1 = 14;  // Left Motor Forward
const int IN2 = 15;  // Left Motor Backward
const int ENA = 11;  // Left Motor Speed (PWM)
const int IN3 = 13;  // Right Motor Forward
const int IN4 = 12;  // Right Motor Backward
const int ENB = 10;  // Right Motor Speed (PWM)

void setup() {
  // Configure motor control and speed pins as outputs
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  pinMode(ENA, OUTPUT);
  pinMode(ENB, OUTPUT);

  // Set direction to Forward once
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);

  // Initialize Serial Monitor for debugging
  Serial.begin(9600);
  Serial.println("Robot Speed Control Initialized.");
}

void loop() {
  // Level 1: Stop (PWM = 0)
  Serial.println("Motor Speed: 0 (Stopped)");
  analogWrite(ENA, 0);
  analogWrite(ENB, 0);
  delay(2000);

  // Level 2: Slow (PWM = 100)
  Serial.println("Motor Speed: 100 (Slow)");
  analogWrite(ENA, 100);
  analogWrite(ENB, 100);
  delay(2000);

  // Level 3: Medium (PWM = 180)
  Serial.println("Motor Speed: 180 (Medium)");
  analogWrite(ENA, 180);
  analogWrite(ENB, 180);
  delay(2000);

  // Level 4: Full Speed (PWM = 255)
  Serial.println("Motor Speed: 255 (Full)");
  analogWrite(ENA, 255);
  analogWrite(ENB, 255);
  delay(2000);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **L298N Motor Driver**, and two **DC Toy Motors** onto the canvas.
2. Wire the L298N VCC to **5V** and GND to **GND**.
3. Remove the jumpers on ENA and ENB on the L298N widget, then connect **ENA** to **GPIO 11** and **ENB** to **GPIO 10**.
4. Wire **IN1** through **IN4** to **GPIO 14, 15, 13, 12** respectively.
5. Connect the motors to the output blocks.
6. Paste the C++ code into the editor.
7. Select **Interpreted Mode** in the simulation dropdown.
8. Click **Run** and watch the motor speed change indicators.

## Expected Output
Serial Monitor:
```
Robot Speed Control Initialized.
Motor Speed: 0 (Stopped)
Motor Speed: 100 (Slow)
Motor Speed: 180 (Medium)
Motor Speed: 255 (Full)
```

## Expected Canvas Behavior
* During Speed 0, the motors stop completely.
* During Speed 100, the virtual DC motors spin slowly.
* During Speed 180, the motors spin at medium speed.
* During Speed 255, the motors spin at maximum speed.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(ENA, OUTPUT)` | Configures the Enable A pin as an output, preparing it for PWM voltage regulation. |
| `digitalWrite(IN1, HIGH)` | Configures the H-bridge channel polarity to run Left Motor in the forward direction. |
| `analogWrite(ENA, 100)` | Generates a PWM wave on GPIO 11 with a duty cycle of ~39% (100 out of 255) to run the motor slowly. |
| `analogWrite(ENB, 255)` | Generates a 100% duty cycle (continuous HIGH) on GPIO 10 to run the motor at full speed. |

## Hardware & Safety Concept
* **PWM Speed Regulation**: DC motor speed is proportional to the average voltage applied across its terminals. Instead of outputting a constant analog voltage (which is inefficient and generates heat), microcontrollers use Pulse Width Modulation (PWM). By pulsing the H-bridge enable lines (ENA/ENB) on and off rapidly, the average voltage delivered to the motor is scaled according to the duty cycle (from 0 for 0V, to 255 for full supply voltage).
* **Motor Dead-Zone**: Small DC motors have a minimum startup voltage (or duty cycle threshold) below which they cannot overcome internal static friction. Usually, a PWM value below 60–80 will result in no movement, causing the motor to hum and draw stall current without rotating. Ensure your code commands speeds above this threshold when driving the robot.

## Try This! (Challenges)
1. **Breathing Speed Cycle**: Write loop-free logic that increments the motor speed by 15 units every 200 ms until it reaches 255, then decrements it back to 0. (Do not use `for` or `while` loops!)
2. **Speed Balance Adjuster**: Real-world motors are rarely identical; one often spins faster than the other, causing the robot to drift. Modify the code to scale down the speed of the faster motor (e.g. `analogWrite(ENB, speed * 0.9)`) to ensure the robot moves straight.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motors hum but do not spin at lower speeds | Speed value is below the motor's starting threshold (dead-zone) | Increase the starting speed command to at least 80 or 100 in your code. |
| Motor speed does not change (always full speed) | The physical jumpers on ENA/ENB are still connected | Make sure to remove the black plastic jumpers from the L298N board's ENA/ENB headers before connecting the GPIO control wires. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [54 - DC Motor Speed Scaling](../intermediate/54-dc-motor-speed-scaling.md)
- [122 - Mobile Robot Left/Right Steering](122-mobile-robot-steering.md)
