# 14 - Pico Dual Button LED

Use two push buttons to independently turn an LED ON and OFF — your first multi-input control project.

## Goal
Learn how to read two separate input pins simultaneously and use them to control a single output, implementing a simple SET/RESET latch pattern in MicroPython.

## What You Will Build
A two-button LED controller:
- **Button A (GP13)**: Turns the LED ON when pressed.
- **Button B (GP14)**: Turns the LED OFF when pressed.
- **LED (GP15)**: Holds its last commanded state (ON or OFF) until the other button is pressed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Tactile Push Button A | `button` | Yes | Yes |
| Tactile Push Button B | `button` | Yes | Yes |
| LED (any colour) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes — for LED current limiting |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Button A | Terminal 1 | GP13 | Blue | Input pin for "LED ON" command |
| Button A | Terminal 2 | GND | Black | Button A shorts GP13 to GND when pressed |
| Button B | Terminal 1 | GP14 | White | Input pin for "LED OFF" command |
| Button B | Terminal 2 | GND | Black | Button B shorts GP14 to GND when pressed |
| LED | Anode (+, longer leg) | GP15 | Red | Output pin driven HIGH/LOW by button commands |
| LED | Cathode (−, shorter leg) | GND | Black | LED return path to ground |
| 330 Ω Resistor | Either leg | In series between GP15 and LED Anode | — | Current limiter — required on hardware |

> **Wiring tip:** Both Button A and Button B use the internal `PULL_UP` resistor — they read HIGH when open, and LOW when pressed (active-low). Both button Terminal 2 connections share the same GND rail. The LED Cathode also connects to this shared GND rail.

## Code
```python
from machine import Pin
import utime

btn_on  = Pin(13, Pin.IN, Pin.PULL_UP)   # Button A: LED ON
btn_off = Pin(14, Pin.IN, Pin.PULL_UP)   # Button B: LED OFF
led     = Pin(15, Pin.OUT)

led_state = False    # Track the current LED state
led.value(0)

print("Dual button control ready.")
print("Button A (GP13) = ON  |  Button B (GP14) = OFF")

while True:
    if btn_on.value() == 0:      # Button A pressed
        if not led_state:
            led_state = True
            led.value(1)
            print("LED → ON")

    if btn_off.value() == 0:     # Button B pressed
        if led_state:
            led_state = False
            led.value(0)
            print("LED → OFF")

    utime.sleep_ms(50)           # 50 ms debounce / polling delay
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two Push Buttons**, and an **LED** onto the canvas.
2. Connect Button A Terminal 1 to **GP13** and Terminal 2 to **GND**.
3. Connect Button B Terminal 1 to **GP14** and Terminal 2 to **GND**.
4. Connect LED Anode to **GP15** and Cathode to **GND**.
5. Paste the code, select **MicroPython** mode, and click **Run**.
6. Click Button A (GP13) to turn the LED ON, then click Button B (GP14) to turn it OFF.

## Expected Output
```
Dual button control ready.
Button A (GP13) = ON  |  Button B (GP14) = OFF
LED → ON
LED → OFF
LED → ON
```

## Expected Canvas Behavior
- Clicking Button A canvas component turns the LED ON and keeps it ON after release.
- Clicking Button B canvas component turns the LED OFF and keeps it OFF after release.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Pin(13, Pin.IN, Pin.PULL_UP)` | Configures GP13 as an active-low input with internal pull-up for Button A. |
| `led_state` | Boolean variable that tracks the current LED state — avoids printing redundant messages when holding a button. |
| `if not led_state: led_state = True` | Only updates state (and prints) when the LED is actually changing state. |
| `utime.sleep_ms(50)` | 50 ms polling delay provides basic button debouncing. |

## Hardware & Safety Concept: SET/RESET Latch
This circuit implements a software **SR latch** (Set-Reset latch): Button A is the SET input (forces output HIGH), Button B is the RESET input (forces output LOW). The output holds its last commanded state indefinitely, regardless of the input buttons' current state. SR latches are widely used in industrial control panels for maintained ON/OFF commands.

## Try This! (Challenges)
1. **Third Button Toggle**: Add a third button on GP12 that toggles the LED regardless of its current state.
2. **State Indicator on Terminal**: Display the current LED state label ("ON" or "OFF") every time the state changes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Both buttons turn LED ON | Button B wired to GP13 | Double-check: Button A → GP13, Button B → GP14. |
| LED flickers on press | Button contact bounce | Increase debounce delay to `sleep_ms(100)`. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [03 - Pico Button LED](03-pico-button-led.md)
- [13 - Pico Button Counter](13-pico-button-counter.md)
