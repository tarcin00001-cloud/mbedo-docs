# 54 - Pico Motor Speed

Adjust the speed of a DC motor using Pulse Width Modulation (PWM) duty cycle control.

## Goal
Learn how to control motor speed by outputting variable PWM duty cycles to the enable pin of an H-bridge driver.

## What You Will Build
A motor speed accelerator:
- **DC Motor (via L298N)**: Accelerates through three distinct speeds: Low (25% power), Medium (50% power), and High (100% power), changing speed every 3 seconds.
- **L298N ENA (GP13)**: Drives the speed control using PWM.
- **IN1 (GP14) & IN2 (GP15)**: Configured as digital outputs to set spin direction.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DC Motor | `dc_motor` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| L298N | ENA | GP13 | Speed PWM control |
| L298N | IN1 | GP14 | Direction control 1 |
| L298N | IN2 | GP15 | Direction control 2 |
| L298N | OUT1 | Motor Pin 1 | Motor channel output |
| L298N | OUT2 | Motor Pin 2 | Motor channel output |
| L298N | GND | GND | Shared ground |

## Code
```cpp
const int ENA = 13; // PWM speed pin
const int IN1 = 14;
const int IN2 = 15;

void setup() {
  pinMode(ENA, OUTPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);

  // Set direction forward
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
}

void loop() {
  // 1. Slow speed (25% duty cycle)
  analogWrite(ENA, 64);
  delay(3000);

  // 2. Medium speed (50% duty cycle)
  analogWrite(ENA, 128);
  delay(3000);

  // 3. Full speed (100% duty cycle)
  analogWrite(ENA, 255);
  delay(3000);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **L298N Driver**, and **DC Motor** onto the canvas.
2. Connect L298N: **ENA** to **GP13** (remove physical jumper first!), **IN1** to **GP14**, **IN2** to **GP15**.
3. Connect L298N output terminals to the motor.
4. Paste code, select the interpreted mode, and click **Run**.
5. Observe the DC motor speed increase in steps.

## Expected Output

Terminal:
```
Simulation active. Driving ENA PWM on GP13.
```

## Expected Canvas Behavior
| Time | ENA (GP13) PWM | Motor Power |
| --- | --- | --- |
| 0–3 s | 64 | 25% (Slow) |
| 3–6 s | 128 | 50% (Medium) |
| 6–9 s | 255 | 100% (Fast) |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `analogWrite(ENA, 128)` | Outputs a 50% duty cycle square wave to GP13, telling the L298N to deliver average half-voltage to the motor. |

## Hardware & Safety Concept: Inductive Kickback
When driving motor speed with PWM, the H-bridge is turned ON and OFF thousands of times per second. During the OFF phases, the motor coil tries to keep current flowing, creating high voltage spikes (inductive kickback). H-bridges like the L298N have built-in protective flyback diodes or require external diodes connected to OUT1/OUT2 to redirect this current safely and prevent chip damage.

## Try This! (Challenges)
1. **Deceleration Cycle**: Modify the loop to step down the speed from Full → Medium → Slow → Stop.
2. **Speed Dial**: Connect a potentiometer on GP26 and map its analog value to control the motor speed dynamically.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor does not move at slow speed | High friction | DC motors need a minimum startup voltage to overcome static friction. If duty cycles below 64 do not turn the motor, increase the lowest PWM value. |

## Mode Notes
This basic PWM speed mapping project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [32 - Pico Potentiometer Dimmer](../../beginner/32-pico-potentiometer-dimmer.md)
- [53 - Pico Motor Toggle](53-pico-motor-toggle.md)
