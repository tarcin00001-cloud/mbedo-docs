# 55 - Pico Motor Direction

Control the rotational direction of a DC motor using an L298N H-Bridge.

## Goal
Learn how to swap voltage polarity across DC motor terminals to control forward and backward rotation.

## What You Will Build
A direction switching cycle:
- **DC Motor**: Rotates forward for 3 seconds, stops for 1 second, rotates backward for 3 seconds, and stops for 1 second.
- **L298N IN1 (GP14) & IN2 (GP15)**: Toggled to swap terminal polarity.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DC Motor | `dc_motor` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| L298N | IN1 | GP14 | Direction control 1 |
| L298N | IN2 | GP15 | Direction control 2 |
| L298N | OUT1 | Motor Pin 1 | Motor channel output |
| L298N | OUT2 | Motor Pin 2 | Motor channel output |
| L298N | GND | GND | Shared ground |

## Code
```cpp
const int IN1 = 14;
const int IN2 = 15;

void setup() {
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
}

void loop() {
  // 1. Spin Forward
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  delay(3000);

  // 2. Coast to Stop
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  delay(1000);

  // 3. Spin Backward
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  delay(3000);

  // 4. Coast to Stop
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **L298N Driver**, and **DC Motor** onto the canvas.
2. Connect L298N: **IN1** to **GP14**, **IN2** to **GP15**, **OUT1/OUT2** to the motor.
3. Paste code, select the interpreted mode, and click **Run**.
4. Observe the DC motor rotation swap direction periodically.

## Expected Output

Terminal:
```
Simulation active. Driving direction toggle.
```

## Expected Canvas Behavior
| Cycle Time | GP14 (IN1) | GP15 (IN2) | Motor Action |
| --- | --- | --- | --- |
| 0–3.0 s | HIGH | LOW | Spinning Forward |
| 3.0–4.0 s | LOW | LOW | Stopped (Coast) |
| 4.0–7.0 s | LOW | HIGH | Spinning Backward |
| 7.0–8.0 s | LOW | LOW | Stopped (Coast) |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(IN1, LOW); digitalWrite(IN2, HIGH);` | Inverts the output polarity: OUT1 goes to GND and OUT2 goes to VCC, reversing current flow through the motor coils. |

## Hardware & Safety Concept: Brake vs. Coast States
When stopping a DC motor via an H-bridge:
- **Coast (Float)**: IN1 = LOW, IN2 = LOW. Motor connections are left open. The motor slows down gradually due to its own momentum and friction.
- **Brake**: IN1 = HIGH, IN2 = HIGH. Motor terminals are shorted to VCC (or GND). The motor generates a back-EMF opposing its own rotation, bringing it to a sudden stop.
Forcing sudden direction changes without stopping in between causes high torque stress on motor gears.

## Try This! (Challenges)
1. **Dynamic Direction switch**: Connect a switch on GP16. If the switch is HIGH, spin forward; if LOW, spin backward.
2. **Double Indicator**: Wire a green LED to GP13 (Forward status) and a red LED to GP12 (Backward status).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor spin direction is reversed | Output terminals swapped | Swap the wires connected to L298N terminals OUT1 and OUT2, or invert the IN1/IN2 pin assignments in your code. |

## Mode Notes
This basic motor control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [53 - Pico Motor Toggle](53-pico-motor-toggle.md)
- [54 - Pico Motor Speed](54-pico-motor-speed.md)
