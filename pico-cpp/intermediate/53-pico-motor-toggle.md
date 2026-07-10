# 53 - Pico Motor Toggle

Start and stop a DC motor using an L298N H-Bridge motor driver.

## Goal
Learn how to interface DC motors with microcontrollers using H-Bridge motor drivers to manage high-current motor loads safely.

## What You Will Build
A motor toggle control:
- **DC Motor (via L298N)**: Alternates between spinning forward for 3 seconds and stopping completely for 3 seconds.
- **L298N IN1 (GP14) & IN2 (GP15)**: Configured as digital outputs to set motor state.

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
| L298N | OUT1 | Motor Pin 1 | Motor driver channel output |
| L298N | OUT2 | Motor Pin 2 | Motor driver channel output |
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
  // Motor Spin Forward
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  delay(3000);

  // Motor Stop
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  delay(3000);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **L298N Driver**, and **DC Motor** onto the canvas.
2. Connect L298N: **IN1** to **GP14**, **IN2** to **GP15**, **OUT1** to **Motor Pin 1**, **OUT2** to **Motor Pin 2**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Observe the motor spinning and stopping in 3-second intervals.

## Expected Output

Terminal:
```
Simulation active. Driving L298N on GP14/15.
```

## Expected Canvas Behavior
| Time | GP14 (IN1) | GP15 (IN2) | Motor Spin State |
| --- | --- | --- | --- |
| 0–3 s | HIGH | LOW | Spinning Forward |
| 3–6 s | LOW | LOW | Stopped |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(IN1, HIGH)` | Sets IN1 HIGH and IN2 LOW to establish a voltage polarity difference across OUT1 and OUT2, making the motor spin. |
| `digitalWrite(IN1, LOW)` | Sets both control pins to LOW to cut current flow and stop the motor. |

## Hardware & Safety Concept: External Motor Power
DC motors require significant voltage and current to operate. You must never power a DC motor directly from the Pico's pins, as this can permanently destroy the RP2040 chip's silicon. Always connect the motor power terminal ($V_M$ / 12V pin) of the L298N module to an external battery or power adapter, keeping the driver's GND connected to the Pico's GND.

## Try This! (Challenges)
1. **Reverse Toggle**: Modify the loop to spin forward for 2 seconds, stop for 1 second, spin backward (IN1 LOW, IN2 HIGH) for 2 seconds, and stop for 1 second.
2. **Indicator Sync**: Wire a green LED to GP13 and program it to turn ON only when the motor is spinning.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor does not spin | Missing enable jumper | Ensure the L298N module's ENA jumper is physically connected (bridging ENA to 5V) to enable the channel. |

## Mode Notes
This basic motor control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [13 - Pico DC Fan Switch](../../beginner/13-pico-dc-fan-switch.md)
- [54 - Pico Motor Speed](54-pico-motor-speed.md)
