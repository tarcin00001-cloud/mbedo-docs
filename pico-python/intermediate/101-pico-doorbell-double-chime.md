# 101 - Pico Doorbell Double Chime

Build a multi-tone doorbell console that sounds a two-tone chime when a button is pressed.

## Goal
Learn how to combine digital inputs (button) with PWM frequency outputs (passive buzzer) to sound multi-tone chimes in MicroPython.

## What You Will Build
A two-tone doorbell:
- **Push Button (GP14)**: Detects door visitor presses.
- **Passive Buzzer (GP15)**: Sounds a high-low chime ("Ding-Dong") when the button is pressed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes (passive required) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button | Terminal 1 | GP14 | Blue | Button signal input (reads LOW on press) |
| Push Button | Terminal 2 | GND | Black | Shorts GP14 to GND on press |
| Passive Buzzer | Signal (+) | GP15 | Red | PWM chime signal |
| Passive Buzzer | GND (−) | GND | Black | Shared return path to ground |

> **Wiring tip:** The button connects between GP14 and GND (using internal pull-up). Connect the passive buzzer (+) pin to GP15 and (-) pin to GND. All grounds are shared.

## Code
```python
from machine import Pin, PWM
import utime

button = Pin(14, Pin.IN, Pin.PULL_UP)

# Set up Passive Buzzer on GP15
buzzer = PWM(Pin(15))

# Ensure buzzer starts silent
buzzer.duty_u16(0)

# Chime frequency pitches (Hertz)
DING_FREQ = 1200 # High note
DONG_FREQ = 800  # Low note

print("Doorbell chime node active.")

while True:
    # Check for button press (active LOW)
    if button.value() == 0:
        print("Ding-Dong! Doorbell pressed.")
        
        # 1. "Ding" Note
        buzzer.freq(DING_FREQ)
        buzzer.duty_u16(32768) # 50% duty cycle
        utime.sleep_ms(250)
        
        # 2. "Dong" Note
        buzzer.freq(DONG_FREQ)
        utime.sleep_ms(400)
        
        # Silence
        buzzer.duty_u16(0)
        
        # Lockout delay to prevent spamming
        utime.sleep(1)
        
    utime.sleep_ms(50) # Polling update speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Push Button**, and **Passive Buzzer** onto the canvas.
2. Connect Button to **GP14** and Buzzer to **GP15**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the button component on the canvas to hear the two-tone chime.

## Expected Output
```
Doorbell chime node active.
Ding-Dong! Doorbell pressed.
```

## Expected Canvas Behavior
- Clicking the button component causes the passive buzzer to activate at alternating frequencies ("Ding" for 250 ms, then "Dong" for 400 ms) before silencing.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `buzzer.freq(DING_FREQ)` | Sets the PWM frequency to 1200 Hz for the high "Ding" tone. |
| `buzzer.duty_u16(32768)` | Opens the PWM channel at 50% duty cycle to play the active note. |
| `buzzer.duty_u16(0)` | Silences the buzzer at the end of the chime sequence. |

## Hardware & Safety Concept: Doorbell Lockout Timers
Mechanical buttons bounce and can be pressed repeatedly by visitors. Doorbell systems include a **lockout delay** (typically 1 to 2 seconds) after a chime completes. This ignores new button presses during the lockout, preventing the buzzer from overheating or sounding annoying chimes continuously.

## Try This! (Challenges)
1. **Indicator Light**: Connect an LED on GP13 and flash it in sync with the chime notes.
2. **Westminster Chime**: Extend the melody to play a longer 4-note chime sequence.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer click/pops without tone | Buzzer is active type | Active buzzers have fixed oscillators and ignore input frequencies. Ensure you use a passive buzzer. |
| Buzzer does not sound | Duty cycle set to 0 | Verify that `buzzer.duty_u16(32768)` is called to start the chime. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [07 - Pico Passive Buzzer Tone](07-pico-tone.md)
- [08 - Pico Button Buzzer](08-pico-button-buzzer.md)
- [20 - Pico Doorbell LED](20-pico-doorbell-led.md)
