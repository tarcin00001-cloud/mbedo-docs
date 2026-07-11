# 122 - Mobile Robot Left/Right Steering

Steer a mobile robot by toggling the states of the left and right DC motors via an L298N motor driver.

## Goal
Understand differential steering kinematics in mobile robotics, control turns by coordinating motor states, and program sequential navigation routines.

## What You Will Build
A steering controller for a two-wheeled mobile robot. The ARIES board drives the left and right motors through the L298N module. By selectively turning motors on, off, or reversing them, the robot executes a sequence: Go Forward (2s), Turn Right (1.5s), Go Forward (2s), Turn Left (1.5s), and Stop (2s).

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
| L298N Module | IN1 | GPIO 14 | Orange | Left motor forward control |
| L298N Module | IN2 | GPIO 15 | Yellow | Left motor reverse control |
| L298N Module | IN3 | GPIO 13 | Green | Right motor forward control |
| L298N Module | IN4 | GPIO 12 | Blue | Right motor reverse control |
| Left DC Motor | Pin 1 / Pin 2 | OUT1 / OUT2 | Red / Black | Left motor output terminals |
| Right DC Motor | Pin 1 / Pin 2 | OUT3 / OUT4 | Red / Black | Right motor output terminals |

> **Wiring tip:** Standard L298N boards have onboard jumpers for ENA and ENB. Ensure these jumpers are plugged in to keep the motor channels enabled, which allows direct control using only the IN1-IN4 pins.

## Code
```cpp
// Mobile Robot Steering Control - VEGA ARIES v3
const int IN1 = 14; // Left Motor Forward
const int IN2 = 15; // Left Motor Backward
const int IN3 = 13; // Right Motor Forward
const int IN4 = 12; // Right Motor Backward

void setup() {
  // Configure motor driver control pins as outputs
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  // Initialize Serial Monitor for debugging
  Serial.begin(9600);
  Serial.println("Mobile Robot Steering Initialized.");
}

void loop() {
  // Phase 1: Go Forward (Both motors forward)
  Serial.println("Direction: Forward");
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  delay(2000);

  // Phase 2: Pivot Turn Right (Left forward, Right backward/stop)
  // Let's do a sharp spin turn: Left motor forward, Right motor backward
  Serial.println("Direction: Sharp Right Turn");
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  delay(1500);

  // Phase 3: Go Forward (Both motors forward)
  Serial.println("Direction: Forward");
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  delay(2000);

  // Phase 4: Pivot Turn Left (Left backward/stop, Right forward)
  // Let's do a sharp spin turn: Left motor backward, Right motor forward
  Serial.println("Direction: Sharp Left Turn");
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  delay(1500);

  // Phase 5: Stop (Both motors off)
  Serial.println("Direction: Stop");
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
  delay(2000);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **L298N Motor Driver**, and two **DC Toy Motors** onto the canvas.
2. Connect the L298N VCC to **5V** and GND to **GND**.
3. Wire the control pins: **IN1** to **GPIO 14**, **IN2** to **GPIO 15**, **IN3** to **GPIO 13**, and **IN4** to **GPIO 12**.
4. Connect the left DC motor to **OUT1/OUT2** and the right DC motor to **OUT3/OUT4**.
5. Paste the C++ code into the editor.
6. Select **Interpreted Mode** in the simulation dropdown.
7. Click **Run** to execute.
8. Observe the motor rotation direction indicators on the canvas.

## Expected Output
Serial Monitor:
```
Mobile Robot Steering Initialized.
Direction: Forward
Direction: Sharp Right Turn
Direction: Forward
Direction: Sharp Left Turn
Direction: Stop
```

## Expected Canvas Behavior
* Forward: Both virtual DC motors rotate in the clockwise direction.
* Right Turn: The Left DC motor rotates clockwise, and the Right DC motor rotates counter-clockwise.
* Left Turn: The Left DC motor rotates counter-clockwise, and the Right DC motor rotates clockwise.
* Stop: Both virtual motors stop spinning.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(IN1, OUTPUT)` | Prepares GPIO 14 to write output signals to the Left forward channel of the H-bridge. |
| `digitalWrite(IN1, HIGH)` | Starts the left motor forward rotation. |
| `digitalWrite(IN4, HIGH)` | Starts the right motor backward rotation during the sharp right turn. |
| `delay(1500)` | Holds the current turning state for 1.5 seconds. |

## Hardware & Safety Concept
* **Differential Steering**: Differential steering controls vehicle motion using two independently driven wheels on opposite sides. By varying the relative speed and direction of the wheels, the robot can move forward, reverse, turn, or spin on the spot. Spinning in place (spin turn) occurs when the wheels rotate in opposite directions at equal speeds, minimizing the turn radius to zero.
* **Current Draw under Sharp Reversals**: Rapidly transitioning a motor from full forward to full reverse (plugging) creates a large transient current spike (stagnation/stall current), which can overload the motor driver and damage it. It is recommended to insert a brief stop state (`digitalWrite(IN1-IN4, LOW)`) for 100-200ms when changing directions on high-torque physical chassis.

## Try This! (Challenges)
1. **Soft Turns**: Modify the code to perform a soft right turn where the left motor runs forward and the right motor is completely turned off (instead of reversed). Compare the turning radius.
2. **Infinite Figure 8**: Program a sequence that continuously navigates the robot in a figure-8 pattern by repeating a loop-free sequence of alternating left and right curves.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The robot turns left when it should turn right | Motor output wires are swapped or code labels do not match physical wiring | Double check which motor is left and which is right. Swap the output wire pins on the L298N terminals or adjust the code pin assignments. |
| The robot moves backward during the forward phase | Both motor wiring pairs are reversed | Swap the terminal connections for both OUT1/OUT2 and OUT3/OUT4 on the L298N driver board. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [121 - Dual Motor Forward/Reverse Drive](121-dual-motor-drive.md)
- [123 - Robot Speed Control](123-robot-speed-control.md)
