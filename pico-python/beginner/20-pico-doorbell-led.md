# 20 - Pico Doorbell LED

Build a smart doorbell console that rings a chime and flashes an indicator LED simultaneously when pressed.

## Goal
Learn how to actuate two different output devices (active buzzer and LED) simultaneously in response to a single digital input signal in MicroPython.

## What You Will Build
An indicator chime doorbell:
- **Push Button (GP14)**: Detects door visitor presses.
- **Active Buzzer (GP15)**: Emits a warning chime on press.
- **LED (GP13)**: Flashes in sync with the buzzer chime to provide visual feedback.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| LED (any colour) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes — for LED current limiting |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button | Terminal 1 | GP14 | Blue | Doorbell button input |
| Push Button | Terminal 2 | GND | Black | Shorts GP14 to GND on press |
| Active Buzzer | VCC (+) | GP15 | Red | Chime output pin |
| Active Buzzer | GND (−) | GND | Black | Shared return path to ground |
| LED | Anode (+, longer leg) | GP13 | Orange | Visual flash signal |
| LED | Cathode (−, shorter leg) | GND | Black | Shared return path to ground |
| 330 Ω Resistor | Either leg | In series between GP13 and LED Anode | — | Limits LED current to ~10 mA |

> **Wiring tip:** The button uses the internal pull-up on GP14. The buzzer and LED cathode share the ground rail on the breadboard. GP15 and GP13 are configured as digital outputs.

## Code
```python
from machine import Pin
import utime

button = Pin(14, Pin.IN, Pin.PULL_UP)
buzzer = Pin(15, Pin.OUT)
led    = Pin(13, Pin.OUT)

# Ensure start state is quiet and dark
buzzer.value(0)
led.value(0)

while True:
    if button.value() == 0: # Visitor press (active LOW)
        print("Ding-Dong! Visitor at the door.")
        
        # Double chime sync loop
        for _ in range(2):
            buzzer.value(1)
            led.value(1)
            utime.sleep_ms(150)
            
            buzzer.value(0)
            led.value(0)
            utime.sleep_ms(100)
            
        # Cooldown lockout to prevent button spamming
        utime.sleep(1)
        
    utime.sleep_ms(50)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Push Button**, **Active Buzzer**, and **LED** onto the canvas.
2. Connect Button to **GP14**, Buzzer to **GP15**, and LED to **GP13**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the button component on the canvas and check the output.

## Expected Output
```
Ding-Dong! Visitor at the door.
```

## Expected Canvas Behavior
- Clicking the button causes both the buzzer and the LED component to pulse twice in synchronization, then pause before resetting.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `buzzer.value(1); led.value(1)` | Driving both pins HIGH simultaneously activates the chime and light indicators together. |
| `range(2)` | Runs a loop twice to generate a classic double-chime beep pattern. |
| `utime.sleep(1)` | Lockout delay to prevent visitors from spamming the chime. |

## Hardware & Safety Concept: Accessibility Visual Feedback
Visual alert feedback is a key requirement in modern access control design. Adding flashing LEDs alongside buzzer chimes ensures accessibility for hearing-impaired users and works well in loud environments where audio alerts might not be heard.

## Try This! (Challenges)
1. **Silent Night Mode**: Add a slide switch on GP12. When ON, disable the buzzer completely and only flash the LED indicator.
2. **Distinctive Ring tones**: Add a second button on GP11 (back door) and configure it to emit a single long beep instead of a double beep.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED flashes but no sound | Buzzer wired backward | Active buzzers are polarized. Ensure the (+) leg connects to GP15 and GND connects to the ground rail. |
| Button triggers continuously | Internal pull-up omitted | Verify the code specifies `Pin.PULL_UP` in the pin configuration. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [03 - Pico Button LED](03-pico-button-led.md)
- [08 - Pico Button Buzzer](08-pico-button-buzzer.md)
