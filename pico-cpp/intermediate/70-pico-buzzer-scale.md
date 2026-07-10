# 70 - Pico Buzzer Scale

Play a basic musical scale chime on a passive buzzer.

## Goal
Learn how to generate different acoustic frequencies (notes) by outputting custom PWM frequencies to a buzzer pin.

## What You Will Build
A musical chime indicator:
- **Passive Buzzer (GP14)**: Plays a sequence of 8 musical notes (do-re-mi scale) from C4 to C5 when the Pico starts up, then remains quiet.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes (piezo transducer) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Passive Buzzer | VCC (+) | GP14 | PWM sound channel |
| Passive Buzzer | GND (-) | GND | Ground return |

## Code
```cpp
const int BUZZER_PIN = 14;

// Frequencies for do-re-mi scale (Hz)
const int C4 = 262;
const int D4 = 294;
const int E4 = 330;
const int F4 = 349;
const int G4 = 392;
const int A4 = 440;
const int B4 = 494;
const int C5 = 523;

void setup() {
  pinMode(BUZZER_PIN, OUTPUT);

  // Play do-re-mi scale once at startup
  tone(BUZZER_PIN, C4, 300); delay(400);
  tone(BUZZER_PIN, D4, 300); delay(400);
  tone(BUZZER_PIN, E4, 300); delay(400);
  tone(BUZZER_PIN, F4, 300); delay(400);
  tone(BUZZER_PIN, G4, 300); delay(400);
  tone(BUZZER_PIN, A4, 300); delay(400);
  tone(BUZZER_PIN, B4, 300); delay(400);
  tone(BUZZER_PIN, C5, 500); delay(600);
}

void loop() {
  // Silent loop
  delay(100);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Passive Buzzer** onto the canvas.
2. Connect Buzzer: **VCC** to **GP14**, **GND** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Listen to the 8-note musical scale chime play on boot.

## Expected Output

Terminal:
```
Simulation active. Playing tone scale on GP14.
```

## Expected Canvas Behavior
* Startup: 8 distinct musical beeps play in ascending frequency.
* Main loop: Silent.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `tone(BUZZER_PIN, C4, 300)` | Generates a 50% duty cycle square wave at 262 Hz on GP14 for a duration of 300 ms. |
| `delay(400)` | Wait time between notes to ensure the previous note finishes playing before starting the next one. |

## Hardware & Safety Concept: Piezo buzzers vs. Magnetic buzzers
- **Piezo Buzzers**: Use a piezoelectric ceramic plate. They have very high impedance, draw very little current (1–5mA), and can be driven directly by the Pico pins without issue.
- **Magnetic (Electromagnetic) Buzzers**: Use a copper coil and magnet. They have low impedance and draw significant current (20–50mA). Driving them directly from microcontroller pins can overload the pin circuits. Always use a transistor switch buffer when driving magnetic sounders.

## Try This! (Challenges)
1. **Descending Scale**: Program the scale to play in reverse order (C5 down to C4) inside the main `loop()` block, repeating every 5 seconds.
2. **Alert Chime**: Create a two-tone alarm pattern (alternating 440 Hz and 880 Hz every 100 ms).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer click but no tone | Active buzzer used | Active buzzers have internal oscillators and will click or beep at a single fixed frequency when driven with `tone()`. Ensure you use a **passive** buzzer for playing melody frequencies. |

## Mode Notes
This basic acoustic tone project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [05 - Pico Active Buzzer](../../beginner/05-pico-active-buzzer.md)
- [06 - Pico Buzzer Patterns](../../beginner/06-pico-buzzer-patterns.md)
