# 08 - Pico Button Buzzer

Build a simple doorbell — press a button and an active buzzer sounds a chime.

## Goal
Learn how to combine a digital input (button) with a digital output (buzzer) so that one device triggers the other, and use a brief timed beep rather than a continuous tone.

## What You Will Build
A doorbell circuit:
- **Push Button (GP14)**: Detects a press.
- **Active Buzzer (GP15)**: Sounds a 300 ms chime when the button is pressed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button | Terminal 1 | GP14 | Blue | Signal input pin — reads LOW when button pressed |
| Push Button | Terminal 2 | GND | Black | Button shorts GP14 to GND on press |
| Active Buzzer | VCC (+) | GP15 | Red | Digital output drives the buzzer ON/OFF |
| Active Buzzer | GND (−) | GND | Black | Buzzer return path to ground |

> **Wiring tip:** The button uses the internal `PULL_UP` resistor — GP14 reads HIGH (3.3 V) when the button is open and LOW (0 V) when pressed. No external pull-up resistor is needed. Both the button Terminal 2 and the buzzer GND pin connect to the same GND rail.

## Code
```python
from machine import Pin
import utime

button = Pin(14, Pin.IN, Pin.PULL_UP)
buzzer = Pin(15, Pin.OUT)

buzzer.value(0)   # Ensure buzzer starts silent

while True:
    if button.value() == 0:    # Button pressed (active LOW)
        buzzer.value(1)        # Sound the chime
        utime.sleep(0.3)       # Chime duration: 300 ms
        buzzer.value(0)        # Silence
        utime.sleep(0.5)       # Lockout: prevent re-trigger for 500 ms
    utime.sleep(0.05)          # 50 ms polling delay
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, a **Push Button**, and an **Active Buzzer** onto the canvas.
2. Connect Button Terminal 1 to **GP14** and Terminal 2 to **GND**.
3. Connect Buzzer VCC (+) to **GP15** and GND (−) to **GND**.
4. Paste the code, select **MicroPython** mode, and click **Run**.
5. Click the button on the canvas to hear the doorbell chime.

## Expected Output
- Button **not pressed**: Buzzer is silent.
- Button **pressed**: Buzzer sounds for 300 ms, then goes silent.

## Expected Canvas Behavior
- Clicking the canvas button triggers the buzzer to activate for 300 ms, then deactivate automatically.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Pin(14, Pin.IN, Pin.PULL_UP)` | Configures GP14 as an input with an internal pull-up, so the pin reads HIGH when unpressed. |
| `button.value() == 0` | Checks if the button has pulled GP14 LOW (i.e. the button is currently pressed). |
| `buzzer.value(1)` | Activates the buzzer by driving GP15 HIGH. |
| `utime.sleep(0.3)` | Holds the chime for exactly 300 ms regardless of how long the button is held. |
| `utime.sleep(0.5)` | Post-chime lockout period — prevents the buzzer from immediately re-triggering if the button is still held. |

## Hardware & Safety Concept: Debouncing and Lockout Periods
Mechanical buttons generate rapid voltage fluctuations (bounce) for a few milliseconds around the moment of contact. Without a lockout delay, a single button press can appear as multiple presses. The 500 ms post-chime delay in this code acts as a simple software lockout, giving the contacts time to settle before the next trigger is checked.

## Try This! (Challenges)
1. **Double Chime**: Make the buzzer beep twice (two short beeps) per press for a classic two-tone doorbell sound.
2. **LED Indicator**: Add an LED on GP13 that flashes in sync with each buzzer beep.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer sounds immediately on startup | Buzzer polarity reversed | Check that the Buzzer VCC (+) is on GP15 (not GND), and GND (−) is on the GND pin. |
| Buzzer triggers multiple times per press | No lockout delay | Add or increase the post-chime `sleep(0.5)` delay after `buzzer.value(0)`. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [03 - Pico Button LED](03-pico-button-led.md)
- [06 - Pico Active Buzzer](06-pico-buzzer.md)
- [15 - Pico SOS Buzzer](15-pico-sos-buzzer.md)
