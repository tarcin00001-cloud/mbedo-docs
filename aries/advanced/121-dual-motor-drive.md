# 121 - Dual Motor Forward/Reverse Drive

Control two DC motors using an L298N H-bridge motor driver to drive a mobile robot chassis forward and in reverse.

## Goal
Learn how an H-bridge motor driver (L298N) works, control DC motor direction using logic inputs, and program coordinated dual-motor movement patterns.

## What You Will Build
A dual-motor drive system. Two DC motors are connected to an L298N motor driver. The ARIES board controls the L298N's input pins (IN1 through IN4) to drive both motors forward for 2 seconds, stop for 1 second, drive in reverse for 2 seconds, and stop for 1 second in a continuous cycle.

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
| L298N Module | VCC (Power) | 5V | Red | Driver logic/motor power (use external power on hardware) |
| L298N Module | GND | GND | Black | Shared ground connection |
| L298N Module | IN1 | GPIO 14 | Orange | Left motor forward control |
| L298N Module | IN2 | GPIO 15 | Yellow | Left motor reverse control |
| L298N Module | IN3 | GPIO 13 | Green | Right motor forward control |
| L298N Module | IN4 | GPIO 12 | Blue | Right motor reverse control |
| Left DC Motor | Pin 1 / Pin 2 | OUT1 / OUT2 | Red / Black | Left motor output terminals |
| Right DC Motor | Pin 1 / Pin 2 | OUT3 / OUT4 | Red / Black | Right motor output terminals |

> **Wiring tip:** When testing on real hardware, connect an external battery pack (e.g. 4x AA batteries, 6V) to the L298N VCC and GND terminals, and connect the ARIES board GND to the L298N GND terminal to ensure a common reference. Do not power DC motors directly from the ARIES 5V pin, as motors can draw high current and cause board resets.

## Code
```cpp
// Dual Motor Drive Control - VEGA ARIES v3
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
  Serial.println("Dual Motor Drive Initialized.");
}

void loop() {
  // Phase 1: Move Forward (IN1=HIGH, IN2=LOW, IN3=HIGH, IN4=LOW)
  Serial.println("Action: Moving Forward");
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  delay(2000);

  // Phase 2: Stop (IN1=LOW, IN2=LOW, IN3=LOW, IN4=LOW)
  Serial.println("Action: Stop");
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
  delay(1000);

  // Phase 3: Move Reverse (IN1=LOW, IN2=HIGH, IN3=LOW, IN4=HIGH)
  Serial.println("Action: Moving Reverse");
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  delay(2000);

  // Phase 4: Stop (IN1=LOW, IN2=LOW, IN3=LOW, IN4=LOW)
  Serial.println("Action: Stop");
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
  delay(1000);
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
8. Observe the motor rotation indicators on the canvas and logging messages in the Serial Monitor.

## Expected Output
Serial Monitor:
```
Dual Motor Drive Initialized.
Action: Moving Forward
Action: Stop
Action: Moving Reverse
Action: Stop
```

## Expected Canvas Behavior
* During the forward phase, both virtual DC motors rotate in the clockwise direction.
* During the stop phases, both motors stop spinning.
* During the reverse phase, both virtual DC motors rotate in the counter-clockwise direction.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(IN1, OUTPUT)` | Configures L298N input control pins as digital outputs. |
| `digitalWrite(IN1, HIGH)` | Drives Left Motor forward by creating a voltage difference across OUT1 and OUT2. |
| `digitalWrite(IN2, LOW)` | Ensures no opposing current flows through Left Motor backward channel. |
| `delay(2000)` | Keeps the current motor state active for 2000 milliseconds (2 seconds). |
| `digitalWrite(IN1, LOW)` | Cuts off power to the motors to bring them to a complete stop. |

## Hardware & Safety Concept
* **H-Bridge Operation**: A DC motor changes direction based on the polarity of the applied voltage. An H-bridge uses four switches (transistors/MOSFETs) to reverse the current flow path through the motor coil. Setting `IN1` HIGH and `IN2` LOW sends current in one direction, while setting `IN1` LOW and `IN2` HIGH reverses the direction. Setting both to LOW (or HIGH) breaks or stops the motor.
* **Inductive Back-EMF Protection**: Turning off a motor causes its magnetic field to collapse, inducing a high-voltage spike (back-EMF) that can destroy digital electronics. The L298N board contains flyback diodes that safely redirect these voltage spikes back to the power supply. Always keep the logic supply separate or buffered when working with higher voltage motors.

## Try This! (Challenges)
1. **Asymmetric Cycles**: Modify the timing so the robot drives forward for 3 seconds but reverses for only 1 second to create an unbalanced movement cycle.
2. **Duty Toggle**: Modify the code so that during reverse, only one of the two motors runs (reverse pivot), while both run during the forward phase.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motors spin in opposite directions during Forward | One of the motor's terminal wires is reversed | Swap the terminal wires for that specific motor on the L298N OUT block (e.g. swap OUT1 and OUT2 wires). |
| Motors do not spin, but Serial prints correctly | Lack of common ground or missing jumper | Ensure the ARIES GND is connected to the L298N GND. Ensure that the ENA and ENB jumpers on the L298N are firmly connected. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [53 - DC Motor Start-Stop](../intermediate/53-dc-motor-start-stop.md)
- [122 - Mobile Robot Left/Right Steering](122-mobile-robot-steering.md)
