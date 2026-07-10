# 05 - Pico Active Buzzer

Drive an active buzzer to sound a repeating alert beep.

## Goal
Learn how to use digital output pins on the Raspberry Pi Pico to control acoustic actuators.

## What You Will Build
An acoustic alert indicator:
- **Active Buzzer (GP14)**: Emits a 500 ms beep tone followed by 500 ms of silence.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Active Buzzer | VCC (positive, marked +) | GP14 | Control output |
| Active Buzzer | GND (negative) | GND | Ground return |

## Code
```cpp
const int BUZZER_PIN = 14;

void setup() {
  pinMode(BUZZER_PIN, OUTPUT);
}

void loop() {
  digitalWrite(BUZZER_PIN, HIGH); // Sound ON
  delay(500);
  digitalWrite(BUZZER_PIN, LOW);  // Sound OFF
  delay(500);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Active Buzzer** onto the canvas.
2. Connect Buzzer positive terminal to **GP14** and negative terminal to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Listen to the repeating beep sound.

## Expected Output

Terminal:
```
Simulation active. GP14 (Buzzer) cycling ON/OFF.
```

## Expected Canvas Behavior
| Time (ms) | Buzzer Pin (GP14) State | Sound Output |
| --- | --- | --- |
| 0–500 | HIGH | Beep tone active |
| 500–1000 | LOW | Silent |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(BUZZER_PIN, HIGH)` | Drives GP14 to 3.3V, powering the internal oscillator of the active buzzer. |

## Hardware & Safety Concept: Active vs. Passive Buzzers
An **active buzzer** has a built-in oscillator. Driving the control pin HIGH is sufficient to produce a fixed-frequency tone. A **passive buzzer** has no built-in oscillator and requires a continuous square wave signal (using `tone()` or PWM) to vibrate the piezoelectric plate and produce sound.

## Try This! (Challenges)
1. **Rapid Chirp**: Shorten the delays to 50 ms to create a rapid warning click.
2. **Doorbell Ring**: Sound the buzzer for 1.5 seconds and then remain silent for 5 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer is silent | Swapped polarity | Buzzers are polarized. Ensure the positive pin (indicated by `+` on the casing) is wired to GP14, not GND. |

## Mode Notes
This basic acoustic project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [06 - Pico Buzzer Patterns](06-pico-buzzer-patterns.md)
- [15 - Pico Buzzer Alarm](15-pico-buzzer-alarm.md)
