# 07 - Pico Passive Buzzer Tone

Drive a passive buzzer using PWM to generate musical notes at specific frequencies.

## Goal
Learn how to use `machine.PWM` to send a square wave at a specific frequency to a passive buzzer, and produce a sequence of musical tones in MicroPython.

## What You Will Build
A melody-playing passive buzzer:
- **Passive Buzzer (GP15)**: Plays a sequence of musical notes using PWM frequency control.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes (must be passive — no internal oscillator) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Passive Buzzer | Signal (+) | GP15 | Red | PWM square wave drives the buzzer membrane |
| Passive Buzzer | GND (−) | GND | Black | Buzzer return path to ground |

> **Wiring tip:** A **passive buzzer** has no internal oscillator — it requires an external frequency signal to vibrate its piezoelectric membrane and produce sound. Connect the signal (+) pin to a PWM-capable GPIO pin (any GP0–GP28 on the Pico).
>
> **Identifying a passive buzzer:** Connect it directly to 3.3 V. If it makes no sound (just a faint click), it is a passive buzzer. If it beeps continuously, it is an active buzzer — use [06 - Pico Active Buzzer](06-pico-buzzer.md) instead.

## Code
```python
from machine import Pin, PWM
import utime

buzzer = PWM(Pin(15))

# Note frequencies in Hz
C4 = 262
D4 = 294
E4 = 330
F4 = 349
G4 = 392
A4 = 440
B4 = 494
C5 = 523

notes = [C4, D4, E4, F4, G4, A4, B4, C5]

def play_note(freq, duration_ms):
    """Play a single note at the given frequency for duration_ms milliseconds."""
    buzzer.freq(freq)
    buzzer.duty_u16(32768)      # 50% duty cycle for maximum volume
    utime.sleep_ms(duration_ms)
    buzzer.duty_u16(0)          # Silence between notes
    utime.sleep_ms(50)          # Brief gap between notes

while True:
    for note in notes:
        play_note(note, 300)
    utime.sleep(1)              # Pause before repeating
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and a **Passive Buzzer** onto the canvas.
2. Connect the Buzzer Signal (+) pin to **GP15** and GND (−) to **GND**.
3. Paste the code, select **MicroPython** mode, and click **Run**.

## Expected Output
The buzzer plays an ascending musical scale (C4 → D4 → E4 → F4 → G4 → A4 → B4 → C5), each note for 300 ms, then pauses 1 second before repeating.

## Expected Canvas Behavior
- The buzzer component activates during each note and silences between notes.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `PWM(Pin(15))` | Attaches the PWM generator to GP15 for frequency-controlled sound output. |
| `buzzer.freq(freq)` | Sets the PWM frequency to match the desired musical note (e.g. 262 Hz for middle C). |
| `buzzer.duty_u16(32768)` | Sets the duty cycle to 50% (32768 / 65535 ≈ 50%) — this maximises the sound volume for a passive buzzer. |
| `buzzer.duty_u16(0)` | Sets the duty cycle to 0%, silencing the buzzer between notes. |

## Hardware & Safety Concept: How Passive Buzzers Work
A passive buzzer contains a piezoelectric disc. When alternating voltage is applied across the disc at an audible frequency, it physically flexes back and forth, displacing air and creating sound waves. The frequency of the PWM signal directly determines the pitch of the note produced.

## Try This! (Challenges)
1. **Play a Song**: Define a list of note/duration pairs and play a recognisable melody (e.g. "Mary Had a Little Lamb").
2. **Button-Controlled Scale**: Wire a button to GP14 — each press advances to the next note in the scale.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| No sound, only a faint click | Active buzzer used | An active buzzer needs the simple ON/OFF approach in [06 - Pico Active Buzzer](06-pico-buzzer.md). |
| Very quiet output | Duty cycle too low | Use `duty_u16(32768)` (50%) for maximum loudness on a passive buzzer. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**. PWM frequency control is supported natively.

## Related Projects
- [06 - Pico Active Buzzer](06-pico-buzzer.md)
- [08 - Pico Button Buzzer](08-pico-button-buzzer.md)
- [15 - Pico SOS Buzzer](15-pico-sos-buzzer.md)
