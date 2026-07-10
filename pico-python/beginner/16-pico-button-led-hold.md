# 16 - Pico Button LED Hold

Light up an LED only while holding down a push button, implementing simple level-state reading.

## Goal
Learn how to read active-low digital inputs in real time without edge latching and drive a digital output directly based on the input state in MicroPython.

## What You Will Build
A direct momentary light control circuit:
- **Push Button (GP14)**: Reads input voltage level (0V when pressed, 3.3V when open).
- **LED (GP15)**: Stays ON as long as the button is held, and turns OFF immediately when released.

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
| Push Button | Terminal 1 | GP14 | Blue | Input signal (reads LOW on press due to internal pull-up) |
| Push Button | Terminal 2 | GND | Black | Shorts GP14 to GND on press |
| LED | Anode (+, longer leg) | GP15 | Red | Signal output pin |
| LED | Cathode (−, shorter leg) | GND | Black | Shared return path to ground |
| 330 Ω Resistor | Either leg | In series between GP15 and LED Anode | — | Limits LED current to ~10 mA |

> **Wiring tip:** By enabling the internal pull-up resistor, the input pin GP14 sits at 3.3V when open. Pressing the button pulls it to 0V (GND). On the breadboard, both the button Terminal 2 and the LED Cathode can connect to the shared ground rail.

## Code
```python
from machine import Pin
import utime

button = Pin(14, Pin.IN, Pin.PULL_UP)
led    = Pin(15, Pin.OUT)

while True:
    # Level-state reading (Direct state check)
    if button.value() == 0:
        led.value(1) # Turn ON
    else:
        led.value(0) # Turn OFF
        
    utime.sleep_ms(20) # Fast polling loop
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Push Button**, and **LED** onto the canvas.
2. Connect Button Terminal 1 to **GP14** and Terminal 2 to **GND**.
3. Connect LED Anode to **GP15** and Cathode to **GND**.
4. Paste the code, select **MicroPython** mode, and click **Run**.
5. Click and hold the button component on the canvas to see the LED respond.

## Expected Output
- Button **held down**: LED is ON.
- Button **released**: LED is OFF.

## Expected Canvas Behavior
- The LED component on the canvas glows only during the periods when the button component is actively clicked.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Pin(14, Pin.IN, Pin.PULL_UP)` | Configures GP14 as an input with internal pull-up, defaulting to 1 (HIGH). |
| `button.value() == 0` | Checks the active state level. If pressed, the pin reads 0 (LOW). |
| `utime.sleep_ms(20)` | Fast polling interval to ensure instantaneous responsiveness. |

## Hardware & Safety Concept: Momentary Controls
Momentary controls are critical in safety systems (such as dead-man switches on heavy machinery or gate openers). If the operator releases the switch, the signal returns to its default state immediately, halting all motors or actuators to prevent injury.

## Try This! (Challenges)
1. **Inverted Operation**: Change the code so the LED remains ON by default and turns OFF only when you press the button.
2. **Double Input Guard**: Add a second button on GP13 and require BOTH buttons to be held down to light up the LED.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays ON continuously | Button wired incorrectly | Ensure the button is wired to GND (Terminal 2) and NOT to 3.3V. |
| LED response lags | Delay in loop is too long | Verify `sleep_ms` is set to a low value (e.g. 20 ms) rather than full seconds. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [03 - Pico Button LED](03-pico-button-led.md)
- [13 - Pico Button Counter](13-pico-button-counter.md)
- [14 - Pico Dual Button LED](14-pico-dual-button-led.md)
