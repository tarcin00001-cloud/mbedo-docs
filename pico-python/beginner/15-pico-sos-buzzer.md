# 15 - Pico SOS Buzzer

Program a buzzer to transmit the international distress signal SOS in Morse code — three short beeps, three long beeps, three short beeps.

## Goal
Learn how to encode structured timing sequences using functions and constants in MicroPython, and apply that structure to generate recognisable patterns on a digital output.

## What You Will Build
An SOS Morse code transmitter:
- **Active Buzzer (GP15)**: Plays the SOS pattern (· · · — — — · · ·) repeatedly with pauses between transmissions.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Active Buzzer | VCC (+) / Signal | GP15 | Red | GPIO drives the buzzer ON/OFF for dot and dash pulses |
| Active Buzzer | GND (−) | GND | Black | Buzzer return path to ground |

> **Wiring tip:** An **active buzzer** sounds whenever its signal pin is driven HIGH — no frequency control is required. The SOS pattern is created entirely by controlling the ON/OFF timing with `utime.sleep_ms()`.
>
> **Hardware mounting:** In a real emergency beacon, the buzzer should be powered from a 5 V supply through a transistor switch (e.g. 2N2222 with a base resistor of 1 kΩ) controlled by the GPIO pin for a louder output. In MbedO simulation and for low-power testing, driving the buzzer directly from 3.3 V GP15 is sufficient.

## Code
```python
from machine import Pin
import utime

buzzer = Pin(15, Pin.OUT)

# Standard Morse code timing (milliseconds)
DOT       = 120     # Duration of a short pulse (dot)
DASH      = 360     # Duration of a long pulse (dash) = 3 × DOT
SYMBOL_GAP = 120    # Gap between symbols within a letter = 1 × DOT
LETTER_GAP = 360    # Gap between letters = 3 × DOT
WORD_GAP   = 840    # Gap between words = 7 × DOT

def beep(duration_ms):
    """Sound the buzzer for the specified duration."""
    buzzer.value(1)
    utime.sleep_ms(duration_ms)
    buzzer.value(0)

def dot():
    beep(DOT)
    utime.sleep_ms(SYMBOL_GAP)

def dash():
    beep(DASH)
    utime.sleep_ms(SYMBOL_GAP)

def send_S():
    """Morse S = · · ·"""
    dot(); dot(); dot()

def send_O():
    """Morse O = — — —"""
    dash(); dash(); dash()

while True:
    print("Transmitting: SOS")

    send_S()
    utime.sleep_ms(LETTER_GAP)

    send_O()
    utime.sleep_ms(LETTER_GAP)

    send_S()
    utime.sleep_ms(WORD_GAP)    # Long pause before repeating
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and an **Active Buzzer** onto the canvas.
2. Connect the Buzzer VCC (+) to **GP15** and GND (−) to **GND**.
3. Paste the code, select **MicroPython** mode, and click **Run**.

## Expected Output
```
Transmitting: SOS
Transmitting: SOS
Transmitting: SOS
```
(Each print corresponds to one full SOS transmission cycle.)

## Expected Canvas Behavior
- The buzzer activates and deactivates in the dot-dash-dot pattern. Short bursts for S (· · ·), longer bursts for O (— — —), then short bursts again for S, followed by a long silence before repeating.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `DOT = 120` | Defines the unit time for Morse code — 120 ms per dot. All other timings are multiples of this. |
| `DASH = 360` | A dash is three times the length of a dot (international Morse standard). |
| `SYMBOL_GAP = 120` | The gap between each dot or dash within the same letter equals one dot duration. |
| `LETTER_GAP = 360` | The gap between letters equals three dot durations. |
| `WORD_GAP = 840` | The gap between full SOS transmissions equals seven dot durations. |
| `beep(duration_ms)` | Helper function that turns the buzzer ON for exactly the specified duration, then OFF. |

## Hardware & Safety Concept: International Morse Code Standard
Morse code timing ratios are standardised by the ITU (International Telecommunication Union): a dash is exactly 3× a dot; gaps between symbols are 1× a dot; gaps between letters are 3× a dot; gaps between words are 7× a dot. SOS (· · · — — — · · ·) was chosen as the universal distress signal because it is easy to recognise and transmit, even by untrained operators.

## Try This! (Challenges)
1. **Full Alphabet**: Extend the code to include a complete Morse alphabet dictionary and spell out your name.
2. **LED Visual Signal**: Add an LED on GP14 that flashes in sync with the buzzer, creating both audio and visual SOS signals.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Pattern sounds like continuous noise | Gaps not long enough | Increase `SYMBOL_GAP` and `LETTER_GAP` to make the dots and dashes more distinct. |
| No sound | Active buzzer polarity reversed | Check that VCC (+) is connected to GP15, not GND. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**. No external libraries are required.

## Related Projects
- [06 - Pico Active Buzzer](06-pico-buzzer.md)
- [07 - Pico Passive Buzzer Tone](07-pico-tone.md)
- [08 - Pico Button Buzzer](08-pico-button-buzzer.md)
