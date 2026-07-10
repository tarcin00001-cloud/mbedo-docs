# 39 - Pico Morse Code Transmitter Button

Build a manual Morse code telegraph key that sounds a buzzer and lights up an LED when you tap a push button.

## Goal
Learn how to poll a digital input pin in real time to control a digital output and a PWM buzzer output simultaneously in MicroPython.

## What You Will Build
A manual telegraph key transmitter:
- **Push Button (GP14)**: Acts as the manual telegraph key.
- **Active Buzzer (GP15)**: Emits a tone when you hold down the button.
- **LED (GP13)**: Lights up in sync with the buzzer to provide visual feedback.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes (telegraph key) |
| Active Buzzer | `buzzer` | Yes | Yes |
| LED (any colour) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button | Terminal 1 | GP14 | Blue | Key input (reads LOW on press) |
| Push Button | Terminal 2 | GND | Black | Shorts GP14 to GND on press |
| Active Buzzer | VCC (+) | GP15 | Red | Audio signal output |
| Active Buzzer | GND (−) | GND | Black | Shared return path to ground |
| LED | Anode (+, longer leg) | GP13 | Orange | Visual signal output |
| LED | Cathode (−, shorter leg) | GND | Black | Shared return path to ground |
| 330 Ω Resistor | Either leg | In series with LED Anode | — | Limits LED current |

> **Wiring tip:** The key button uses the internal pull-up on GP14. The buzzer and LED cathode share the ground rail. The buzzer signal pin connects to GP15, and the LED anode connects to GP13.

## Code
```python
from machine import Pin
import utime

key_btn = Pin(14, Pin.IN, Pin.PULL_UP)
buzzer  = Pin(15, Pin.OUT)
led     = Pin(13, Pin.OUT)

# Start state
buzzer.value(0)
led.value(0)

print("Morse Code Telegraph Key Ready. Tap the button!")

while True:
    # Direct level check
    if key_btn.value() == 0:
        buzzer.value(1) # Sound ON
        led.value(1)    # Light ON
    else:
        buzzer.value(0) # Sound OFF
        led.value(0)    # Light OFF
        
    utime.sleep_ms(10) # Minimal delay for responsive tapping
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Push Button**, **Active Buzzer**, and **LED** onto the canvas.
2. Connect Button to **GP14**, Buzzer to **GP15**, and LED to **GP13**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click and hold the button component on the canvas to send Morse code signals.

## Expected Output
```
Morse Code Telegraph Key Ready. Tap the button!
```

## Expected Canvas Behavior
- The buzzer and LED components on the canvas activate and turn ON only during the exact periods you click and hold the button component.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `key_btn.value() == 0` | Reads the state of the button (0 = pressed). |
| `utime.sleep_ms(10)` | Low polling delay to ensure maximum responsiveness when typing fast Morse dots and dashes. |

## Hardware & Safety Concept: Morse Code Telegraph Keys
Telegraph keys are simple momentary switches that connect a power source to a transmitter circuit when pressed. Modern training boards pair these keys with a local buzzer and LED to allow operators to practice sending message patterns (like dots and dashes) before connecting to real radio equipment.

## Try This! (Challenges)
1. **Speed Test Datalogger**: Print the duration of each dot or dash (the time the button was held down) to the serial terminal.
2. **Frequency Control**: Replace the active buzzer with a passive buzzer and change its pitch based on how hard or fast you tap the key.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm sounds continuously | Button pin wired incorrectly | Verify the button connects to GND and NOT to 3.3V. |
| Key presses feel laggy or delayed | Polling delay in loop is too long | Ensure `sleep_ms` is set to 10 ms or lower for real-time responsiveness. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [03 - Pico Button LED](03-pico-button-led.md)
- [08 - Pico Button Buzzer](08-pico-button-buzzer.md)
- [20 - Pico Doorbell LED](20-pico-doorbell-led.md)
