# 03 - Pico Button LED

Press a push button to toggle an LED ON and OFF — your first interactive input/output project.

## Goal
Learn how to configure a GPIO pin as a digital input with an internal pull-up resistor, read its state, and respond by controlling an output pin.

## What You Will Build
An interactive button-controlled LED:
- **Push Button (GP14)**: Reads the button state (pressed = LOW due to pull-up).
- **LED (GP15)**: Turns ON while the button is pressed, OFF when released.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| LED (any colour) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes — for LED current limiting |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button | Terminal 1 | GP14 | Blue | Signal input pin (reads HIGH when open, LOW when pressed) |
| Push Button | Terminal 2 | GND | Black | Button shorts GP14 to GND when pressed |
| LED | Anode (+, longer leg) | GP15 | Red | Signal output pin |
| LED | Cathode (−, shorter leg) | GND | Black | LED return path to ground |
| 330 Ω Resistor | Either leg | In series between GP15 and LED Anode | — | Limits LED current to ~10 mA |

> **Wiring tip:** The internal pull-up resistor (`Pin.PULL_UP`) keeps GP14 at 3.3 V (HIGH) when the button is open. Pressing the button connects GP14 directly to GND, pulling the reading LOW. No external resistor is needed for the button.

## Code
```python
from machine import Pin
import utime

button = Pin(14, Pin.IN, Pin.PULL_UP)
led    = Pin(15, Pin.OUT)

while True:
    if button.value() == 0:   # Button pressed (active LOW)
        led.value(1)          # Turn LED ON
    else:
        led.value(0)          # Turn LED OFF
    utime.sleep(0.05)         # 50 ms debounce delay
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, a **Push Button**, and an **LED** onto the canvas.
2. Connect Button Terminal 1 to **GP14** and Terminal 2 to **GND**.
3. Connect LED Anode to **GP15** and Cathode to **GND**.
4. Paste the code, select **MicroPython** mode, and click **Run**.
5. Click the button on the canvas and watch the LED respond.

## Expected Output
- Button **not pressed**: LED is OFF.
- Button **pressed**: LED is ON.

## Expected Canvas Behavior
- Clicking the canvas button component immediately turns the LED component ON. Releasing it turns it OFF.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Pin(14, Pin.IN, Pin.PULL_UP)` | Configures GP14 as an input with the internal pull-up resistor enabled, so it reads HIGH when unpressed. |
| `button.value() == 0` | Returns `True` when the button is pressed (GP14 pulled LOW by the GND connection). |
| `led.value(1)` | Turns the LED ON by driving GP15 HIGH. |
| `utime.sleep(0.05)` | Short polling delay to debounce mechanical button contacts. |

## Hardware & Safety Concept: Pull-Up Resistors
Without a pull-up resistor, an unconnected input pin floats at an undefined voltage — it may read random HIGH or LOW values, causing unpredictable behaviour. The internal `PULL_UP` connects a ~50 kΩ resistor internally between the pin and 3.3 V, keeping it at a stable HIGH when the button is open.

## Try This! (Challenges)
1. **Toggle Mode**: Instead of holding, make a single button press toggle the LED state permanently until pressed again.
2. **Blink on Press**: Make the LED blink rapidly while the button is held, and stop when released.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED always ON | Button wired to 3.3 V instead of GND | The button Terminal 2 must connect to GND, not 3.3 V. |
| LED flickers when button pressed | Contact bounce | Increase the sleep delay to `0.1` (100 ms) for better debouncing. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**. No compilation step is needed.

## Related Projects
- [08 - Pico Button Buzzer](08-pico-button-buzzer.md)
- [13 - Pico Button Counter](13-pico-button-counter.md)
- [14 - Pico Dual Button LED](14-pico-dual-button-led.md)
