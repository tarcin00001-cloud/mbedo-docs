# 38 - Pico Morse Code Buzzer

Transmit the international distress beacon SOS (· · · — — — · · ·) audibly using a passive buzzer.

## Goal
Learn how to use PWM frequency generation to emit clear audio tones, and implement a structured timing sequencer in MicroPython.

## What You Will Build
An audio emergency beacon:
- **Passive Buzzer (GP15)**: Emits the Morse code SOS pattern at a clean 1000 Hz pitch frequency.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes (passive required) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Passive Buzzer | Signal (+) | GP15 | Red | PWM frequency output pin |
| Passive Buzzer | GND (−) | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the passive buzzer (+) pin to GP15, and (-) pin to GND. Passive buzzers require a pulsing signal to generate sound.

## Code
```python
from machine import Pin, PWM
import utime

buzzer = PWM(Pin(15))
buzzer.freq(1000) # 1 kHz tone (clear pitch)

# Timing constants (in milliseconds)
DOT        = 120
DASH       = 360 # 3x DOT
GAP        = 120 # Gap between dots/dashes
LETTER_GAP = 360 # Gap between letters
WORD_GAP   = 840 # Gap before repeating

def beep(duration):
    buzzer.duty_u16(32768) # 50% duty cycle (Sound ON)
    utime.sleep_ms(duration)
    buzzer.duty_u16(0)     # Sound OFF
    utime.sleep_ms(GAP)

def send_s():
    # S = dot, dot, dot
    beep(DOT)
    beep(DOT)
    beep(DOT)

def send_o():
    # O = dash, dash, dash
    beep(DASH)
    beep(DASH)
    beep(DASH)

print("Starting audio SOS beacon.")

while True:
    send_s()
    utime.sleep_ms(LETTER_GAP)
    
    send_o()
    utime.sleep_ms(LETTER_GAP)
    
    send_s()
    utime.sleep_ms(WORD_GAP) # Cooldown before next sequence
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Passive Buzzer** onto the canvas.
2. Connect Buzzer Signal (+) to **GP15** and GND (−) to **GND**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Observe the audio sequence.

## Expected Output
```
Starting audio SOS beacon.
```

## Expected Canvas Behavior
- The passive buzzer component activates and deactivates in the dot-dash-dot rhythm (short bursts for S, longer bursts for O, then short bursts again for S, followed by a long silence).

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `buzzer.duty_u16(32768)` | Switches the buzzer ON by setting a 50% square wave duty cycle. |
| `buzzer.duty_u16(0)` | Silences the buzzer between pulses by driving the duty cycle to 0%. |

## Hardware & Safety Concept: Morse Code Pitch Selection
Distress beacons typically use tone pitches between **800 Hz and 1200 Hz**. This frequency range matches the human ear's peak sensitivity, helping the warning signal cut through environmental noise and static over long-distance radio transmissions.

## Try This! (Challenges)
1. **Visual Flash**: Connect an LED on GP13 and configure it to flash in sync with the buzzer beeps.
2. **Frequency Shift**: Change the tone pitch to a higher frequency (e.g. 1500 Hz) to test how it sounds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer is silent | Buzzer is active type | Active buzzers require simple ON/OFF signals. Verify you are using a passive buzzer or adapt the code to write simple HIGH/LOW values. |
| Beep gaps are missing | `GAP` timing omitted in `beep()` | Ensure the `beep()` function contains the trailing `sleep_ms(GAP)` call. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [07 - Pico Passive Buzzer Tone](07-pico-tone.md)
- [15 - Pico SOS Buzzer](15-pico-sos-buzzer.md)
- [21 - Pico Morse Code LED](21-pico-morse-code-led.md)
