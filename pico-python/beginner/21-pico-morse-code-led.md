# 21 - Pico Morse Code LED

Transmit the international distress beacon SOS (· · · — — — · · ·) visually using an LED.

## Goal
Learn how to encode complex timing arrays (dots and dashes) and implement them to toggle a single digital output channel in MicroPython.

## What You Will Build
A visual emergency beacon:
- **LED (GP15)**: Pulses the Morse code SOS pattern repeatedly.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LED (any colour) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes — for LED current limiting |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED | Anode (+, longer leg) | GP15 | Red | Signal output pin |
| LED | Cathode (−, shorter leg) | GND | Black | Return path to ground |
| 330 Ω Resistor | Either leg | In series between GP15 and LED Anode | — | Limits LED current to ~10 mA |

> **Wiring tip:** Connect the LED anode to GP15 through the current-limiting resistor, and the cathode to a Pico ground (GND) pin.

## Code
```python
from machine import Pin
import utime

led = Pin(15, Pin.OUT)

# Standard Morse code timing constants (in milliseconds)
DOT       = 150    # Short pulse duration
DASH      = 450    # Long pulse duration (3x DOT)
GAP       = 150    # Element spacing (1x DOT)
LETTER    = 450    # Letter spacing (3x DOT)
WORD      = 1050   # Word spacing (7x DOT)

def flash(duration):
    led.value(1)
    utime.sleep_ms(duration)
    led.value(0)
    utime.sleep_ms(GAP)

def send_s():
    # S = dot, dot, dot
    flash(DOT)
    flash(DOT)
    flash(DOT)

def send_o():
    # O = dash, dash, dash
    flash(DASH)
    flash(DASH)
    flash(DASH)

while True:
    print("Flashing: S O S")
    
    send_s()
    utime.sleep_ms(LETTER)
    
    send_o()
    utime.sleep_ms(LETTER)
    
    send_s()
    utime.sleep_ms(WORD) # Cooldown gap between loops
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **LED** onto the canvas.
2. Connect LED Anode to **GP15** and Cathode to **GND**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Observe the flash intervals.

## Expected Output
```
Flashing: S O S
Flashing: S O S
```

## Expected Canvas Behavior
- The LED component on the canvas flashes in the structured dot-dash rhythm, pausing before repeating.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `flash(duration)` | Helper function that handles turning the LED ON, waiting, turning it OFF, and inserting the inter-element gap. |
| `DASH = 450` | A dash is defined as exactly three times the duration of a dot in standard Morse code. |
| `utime.sleep_ms(WORD)` | Creates a longer pause at the end of the word before repeating the transmission. |

## Hardware & Safety Concept: Morse Code Timing Standards
Standard Morse code uses a **1:3 timing ratio**: a dash is three times longer than a dot. The gap between elements is one dot, the gap between letters is three dots, and the gap between words is seven dots. Maintaining these precise ratios ensures the message remains legible to human operators or receiver software.

## Try This! (Challenges)
1. **Audio Backup**: Add a buzzer on GP14 and configure it to sound in sync with the LED flashes.
2. **Speed Knob**: Connect a potentiometer on GP26 to adjust the base `DOT` rate, making the SOS sequence faster or slower.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Flash is erratic or lacks structure | Gap timing omitted | Ensure the `flash()` helper function includes the post-flash `utime.sleep_ms(GAP)` call. |
| LED stays OFF | Incorrect pin | Verify the LED is wired to GP15 as configured in the code. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [01 - Pico LED Blink](01-pico-blink.md)
- [15 - Pico SOS Buzzer](15-pico-sos-buzzer.md)
