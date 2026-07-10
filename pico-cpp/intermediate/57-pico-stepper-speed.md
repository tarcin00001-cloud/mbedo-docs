# 57 - Pico Stepper Speed

Control the rotational speed of a stepper motor using a potentiometer.

## Goal
Learn how to map analog inputs to variable delays to adjust stepper motor stepping frequency.

## What You Will Build
An adjustable speed stepper drive:
- **Potentiometer (GP26)**: Reads 0 to 4095.
- **Stepper Motor (via ULN2003)**: Step rate scales dynamically. Higher analog voltages decrease the step delay, speeding up rotation.
- **ULN2003 Control (GP12, GP13, GP14, GP15)**: Drives stepper phases.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Stepper Motor | `stepper` | Yes | Yes |
| ULN2003 Driver Module | `uln2003` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Potentiometer | Pin 1 (VCC) | 3.3V | Power supply |
| Potentiometer | Pin 2 (Wiper) | GP26 | Analog input |
| Potentiometer | Pin 3 (GND) | GND | Ground return |
| ULN2003 | IN1 | GP12 | Phase A control |
| ULN2003 | IN2 | GP13 | Phase B control |
| ULN2003 | IN3 | GP14 | Phase C control |
| ULN2003 | IN4 | GP15 | Phase D control |
| ULN2003 | GND | GND | Ground reference |

## Code
```cpp
const int POT_PIN = 26;
const int IN1 = 12;
const int IN2 = 13;
const int IN3 = 14;
const int IN4 = 15;

int stepIndex = 0;

void setup() {
  pinMode(POT_PIN, INPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
}

void loop() {
  int rawValue = analogRead(POT_PIN); // Reads 0 to 4095

  // Map 12-bit ADC (0-4095) to step delay range (3 ms to 50 ms)
  // Higher analog input = lower delay = faster speed
  int stepDelay = 50 - (rawValue * 47 / 4095);

  // Step sequence
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

  // Shift phases
  stepIndex = stepIndex + 1;
  if (stepIndex > 3) {
    stepIndex = 0;
  }

  delay(stepDelay); // Apply dynamic delay
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **ULN2003 Driver**, **Stepper Motor**, and **Potentiometer** onto the canvas.
2. Connect Potentiometer: **Pin 1** to **3V3**, **Pin 2** to **GP26**, **Pin 3** to **GND**.
3. Connect Driver: **IN1-IN4** to **GP12-GP15**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Adjust the potentiometer and watch the stepper rotation speed scale.

## Expected Output

Terminal:
```
Simulation active. Driving variable delay stepper controller.
```

## Expected Canvas Behavior
| Potentiometer Position | GP26 Input Value | Step Delay Time | Shaft Rotation Speed |
| --- | --- | --- | --- |
| Far Left | 0 | 50 ms | Very Slow |
| Center | 2048 | ~26 ms | Medium |
| Far Right | 4095 | 3 ms | Fast |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `50 - (rawValue * 47 / 4095)` | Maps the range such that maximum analog input (4095) results in the lowest delay limit (3 ms), yielding maximum speed. |

## Hardware & Safety Concept: Stepper Pull-in Torque
Stepper motors have a physical limit to how fast they can start from a standstill (called the **pull-in rate**). If you attempt to step the motor at maximum speed immediately (e.g. 3 ms delay) without acceleration ramps, the shaft will merely vibrate and fail to turn. For high-speed operations, software must gradually ramp up speed (decrease delays) to overcome inertia.

## Try This! (Challenges)
1. **Direction Switch**: Connect a button to GP16. If pressed, invert the step sequence to swap rotation direction.
2. **Speed Indicator**: Wire a blue LED to GP11 and flash it in sync with the step rate.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor stalls at max speed | Delay is set too low | Verify the minimum delay value does not drop below 2–3 ms, which is the mechanical speed limit of the 28BYJ-48 stepper. |

## Mode Notes
This basic analog mapping project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [31 - Pico Potentiometer ADC](../../beginner/31-pico-potentiometer-adc.md)
- [56 - Pico Stepper Steps](56-pico-stepper-steps.md)
