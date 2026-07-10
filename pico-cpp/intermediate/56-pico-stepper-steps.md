# 56 - Pico Stepper Steps

Control a 28BYJ-48 stepper motor's step sequence using a ULN2003 driver.

## Goal
Learn how to drive unipolar stepper motors by toggling 4 phase output lines sequentially to rotate the motor shaft by precise angles.

## What You Will Build
A step sequencer:
- **Stepper Motor (via ULN2003)**: Performs a continuous full-step sequence to rotate the shaft in one direction.
- **ULN2003 Inputs (GP12, GP13, GP14, GP15)**: Configured as digital outputs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Stepper Motor | `stepper` | Yes | Yes |
| ULN2003 Driver Module | `uln2003` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| ULN2003 | IN1 | GP12 | Phase A control |
| ULN2003 | IN2 | GP13 | Phase B control |
| ULN2003 | IN3 | GP14 | Phase C control |
| ULN2003 | IN4 | GP15 | Phase D control |
| ULN2003 | GND | GND | Ground reference |

## Code
```cpp
const int IN1 = 12;
const int IN2 = 13;
const int IN3 = 14;
const int IN4 = 15;

int stepIndex = 0;

void setup() {
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
}

void loop() {
  // Step sequence: single phase active (Wave drive)
  if (stepIndex == 0) {
    digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  digitalWrite(IN3, LOW);  digitalWrite(IN4, LOW);
  }
  if (stepIndex == 1) {
    digitalWrite(IN1, LOW);  digitalWrite(IN2, HIGH); digitalWrite(IN3, LOW);  digitalWrite(IN4, LOW);
  }
  if (stepIndex == 2) {
    digitalWrite(IN1, LOW);  digitalWrite(IN2, LOW);  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
  }
  if (stepIndex == 3) {
    digitalWrite(IN1, LOW);  digitalWrite(IN2, LOW);  digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH);
  }

  // Increment step index and wrap around (0 to 3)
  stepIndex = stepIndex + 1;
  if (stepIndex > 3) {
    stepIndex = 0;
  }

  delay(10); // Controls stepper rotation speed
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **ULN2003 Driver**, and **Stepper Motor** onto the canvas.
2. Connect Driver: **IN1** to **GP12**, **IN2** to **GP13**, **IN3** to **GP14**, **IN4** to **GP15**.
3. Connect driver outputs to the stepper motor connector.
4. Paste code, select the interpreted mode, and click **Run**.
5. Observe the stepper motor shaft turning smoothly.

## Expected Output

Terminal:
```
Simulation active. Driving ULN2003 stepper phases.
```

## Expected Canvas Behavior
| Step Index | Phase A (GP12) | Phase B (GP13) | Phase C (GP14) | Phase D (GP15) |
| --- | --- | --- | --- | --- | --- |
| 0 | HIGH | LOW | LOW | LOW |
| 1 | LOW | HIGH | LOW | LOW |
| 2 | LOW | LOW | HIGH | LOW |
| 3 | LOW | LOW | LOW | HIGH |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `stepIndex = stepIndex + 1` | Increments the step phase sequence, wrapping back to 0 once 3 is exceeded. |
| `delay(10)` | Pause time between steps. Slower delays increase shaft holding torque but limit max speed. |

## Hardware & Safety Concept: Unipolar Stepper Motors
The 28BYJ-48 is a unipolar stepper motor containing 4 coils sharing a common center-tap power connection (5V). Driving the motor requires pulling each coil to ground in sequence. Since microcontroller pins cannot sink the required coil current (around 100mA each), a **Darlington transistor array** like the ULN2003 is placed in between to act as low-side switches.

## Try This! (Challenges)
1. **Reverse Direction**: Invert the step sequence logic so the motor rotates in the opposite direction.
2. **Double Phase Drive**: Modify the step logic to drive two phases HIGH simultaneously (e.g. 1 & 2, then 2 & 3, then 3 & 4) to increase motor torque.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor vibrates but doesn't turn | Out-of-order phase wiring | Confirm that GP12-GP15 are wired sequentially to IN1-IN4. If phase connections are swapped, the fields fight each other. |

## Mode Notes
This basic stepper control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [53 - Pico Motor Toggle](53-pico-motor-toggle.md)
- [57 - Pico Stepper Speed](57-pico-stepper-speed.md)
