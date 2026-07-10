# 15 - Pico Buzzer Alarm

Sound a high-frequency continuous warning alarm using an active buzzer.

## Goal
Learn how to program continuous warning indicators using digital output signals on the Raspberry Pi Pico.

## What You Will Build
A security/safety siren:
- **Active Buzzer (GP14)**: Sounds a rapid, high-intensity oscillating beep pattern (250 ms ON, 250 ms OFF) representing a safety hazard trigger.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Active Buzzer | VCC (+) | GP14 | Buzzer control pin |
| Active Buzzer | GND (-) | GND | Ground return |

## Code
```cpp
const int BUZZER_PIN = 14;

void setup() {
  pinMode(BUZZER_PIN, OUTPUT);
}

void loop() {
  // Rapid alert pulse
  digitalWrite(BUZZER_PIN, HIGH);
  delay(250);
  digitalWrite(BUZZER_PIN, LOW);
  delay(250);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Active Buzzer** onto the canvas.
2. Connect Buzzer: positive terminal (+) to **GP14**, negative terminal to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Listen to the high-frequency oscillating warning alarm.

## Expected Output

Terminal:
```
Simulation active. GP14 (Alarm Buzzer) cycling ON/OFF at 250ms intervals.
```

## Expected Canvas Behavior
| Cycle Time | Buzzer Pin (GP14) | Alarm Sound |
| --- | --- | --- |
| 0–250 ms | HIGH | ON (Siren sound) |
| 250–500 ms | LOW | OFF (Silent) |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `delay(250)` | Controls the warning oscillation speed, creating a more urgent beep rate than standard status indicators. |

## Hardware & Safety Concept: Current Limits on Sounders
Buzzer elements pull significant current during oscillator operation. Although miniature buzzers can be driven directly by the Pico's GPIO pins, larger industrial sirens require high-current drivers (like NPN transistors or MOSFET gates) to isolate the microcontroller from the load current.

## Try This! (Challenges)
1. **Accelerating Alarm**: Modify the delay parameters to make the beeps get progressively faster (e.g. starting at 500 ms and decreasing to 50 ms).
2. **Alert Sync**: Connect an external Red LED to GP15 and program it to flash in sync with the buzzer alarm.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer sounds weak or scratchy | Loose ground | Check the connection from the negative pin to the Pico's GND pin to ensure a solid return path. |

## Mode Notes
This basic acoustic warning project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [05 - Pico Active Buzzer](05-pico-active-buzzer.md)
- [06 - Pico Buzzer Patterns](06-pico-buzzer-patterns.md)
