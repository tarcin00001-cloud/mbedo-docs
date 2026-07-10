# 06 - Pico Buzzer Patterns

Generate complex warning sound patterns using an active buzzer to indicate different alarm states.

## Goal
Learn how to program sequential timing loops with varying active and inactive states to create distinct sound notifications.

## What You Will Build
A triple-chirp warning alarm:
- **Alert Pattern**: Sounds 3 quick chirps (100 ms on, 100 ms off) followed by a 1-second pause.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Active Buzzer | VCC (+) | GP14 | Control output |
| Active Buzzer | GND (-) | GND | Ground return |

## Code
```cpp
const int BUZZER_PIN = 14;

void setup() {
  pinMode(BUZZER_PIN, OUTPUT);
}

void loop() {
  // Chirp 1
  digitalWrite(BUZZER_PIN, HIGH);
  delay(100);
  digitalWrite(BUZZER_PIN, LOW);
  delay(100);

  // Chirp 2
  digitalWrite(BUZZER_PIN, HIGH);
  delay(100);
  digitalWrite(BUZZER_PIN, LOW);
  delay(100);

  // Chirp 3
  digitalWrite(BUZZER_PIN, HIGH);
  delay(100);
  digitalWrite(BUZZER_PIN, LOW);
  delay(1000); // 1-second break
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Active Buzzer** onto the canvas.
2. Connect Buzzer positive terminal to **GP14** and negative terminal to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Listen to the distinct triple-chirp warning pattern.

## Expected Output

Terminal:
```
Simulation active. Repeating triple-chirp warning pattern.
```

## Expected Canvas Behavior
| Cycle Time | Buzzer State | Output State |
| --- | --- | --- |
| 0–100 ms | HIGH | Chirp 1 |
| 100–200 ms | LOW | Silence |
| 200–300 ms | HIGH | Chirp 2 |
| 300–400 ms | LOW | Silence |
| 400–500 ms | HIGH | Chirp 3 |
| 500–1500 ms | LOW | Long Pause |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `delay(1000)` | Holds the buzzer in a LOW state for 1 second, demarcating the end of one alarm cycle. |

## Hardware & Safety Concept: Sound Patterns
Different sound patterns represent different alarm categories in safety systems (e.g. slow pulses indicate gas leaks, rapid chirps indicate proximity alerts, and steady tones indicate fire). Standardizing sound patterns helps operators react correctly without looking at screens.

## Try This! (Challenges)
1. **Siren Sweep**: Alternately pulse the buzzer at 300 ms intervals and then at 100 ms intervals.
2. **Double Beep**: Program a pattern of two 200 ms beeps separated by 200 ms, followed by a 2-second pause.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sound lacks tone | Weak digital driving voltage | Ensure the buzzer negative wire is firmly connected to Pico's physical GND pin. |

## Mode Notes
This basic acoustic sequencer runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [05 - Pico Active Buzzer](05-pico-active-buzzer.md)
- [15 - Pico Buzzer Alarm](15-pico-buzzer-alarm.md)
