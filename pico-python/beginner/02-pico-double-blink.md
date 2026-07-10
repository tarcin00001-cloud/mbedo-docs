# 02 - Pico Double Blink

Drive two LEDs in an alternating chase pattern — one turns ON while the other turns OFF.

## Goal
Learn how to control multiple GPIO output pins simultaneously and create simple alternating blink patterns using MicroPython.

## What You Will Build
An alternating two-LED circuit:
- **LED 1 (GP14)**: ON while LED 2 is OFF.
- **LED 2 (GP15)**: ON while LED 1 is OFF.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LED 1 (Red) | `led` | Yes | Yes |
| LED 2 (Green) | `led` | Yes | Yes |
| 330 Ω Resistors × 2 | `resistor` | Optional in MbedO | Yes — one resistor per LED |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED 1 | Anode (+, longer leg) | GP14 | Red | Signal pin for LED 1 |
| LED 1 | Cathode (−, shorter leg) | GND | Black | Shared ground rail |
| 330 Ω Resistor 1 | Either leg | In series between GP14 and LED 1 Anode | — | Current limiter ~10 mA |
| LED 2 | Anode (+, longer leg) | GP15 | Yellow | Signal pin for LED 2 |
| LED 2 | Cathode (−, shorter leg) | GND | Black | Shared ground rail |
| 330 Ω Resistor 2 | Either leg | In series between GP15 and LED 2 Anode | — | Current limiter ~10 mA |

> **Wiring tip:** Both LED cathodes can share the same GND rail on a breadboard. Place each 330 Ω resistor between the GPIO pin and the LED anode.

## Code
```python
from machine import Pin
import utime

led1 = Pin(14, Pin.OUT)
led2 = Pin(15, Pin.OUT)

while True:
    led1.value(1)      # LED 1 ON
    led2.value(0)      # LED 2 OFF
    utime.sleep(0.5)

    led1.value(0)      # LED 1 OFF
    led2.value(1)      # LED 2 ON
    utime.sleep(0.5)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **two LEDs** onto the canvas.
2. Connect LED 1 Anode to **GP14** and LED 2 Anode to **GP15**; connect both Cathodes to **GND**.
3. Paste the code, select **MicroPython** mode, and click **Run**.

## Expected Output
LED 1 glows for 0.5 s while LED 2 is off, then they swap — repeating continuously.

## Expected Canvas Behavior
- LED 1 and LED 2 alternately glow and dim in a chasing pattern every half second.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Pin(14, Pin.OUT)` | Configures GP14 as a digital output for LED 1. |
| `Pin(15, Pin.OUT)` | Configures GP15 as a digital output for LED 2. |
| `led1.value(1); led2.value(0)` | Sets LED 1 HIGH and LED 2 LOW simultaneously in the same half-cycle. |
| `utime.sleep(0.5)` | Holds the current state for half a second before swapping. |

## Hardware & Safety Concept: Shared Ground Rail
On a breadboard, it is common practice to connect all component cathodes (GND legs) to a single shared ground rail. This reduces wiring clutter and ensures all components share the same reference voltage. Always verify that the ground rail is connected to a GND pin on the Pico.

## Try This! (Challenges)
1. **Three-LED Chase**: Add a third LED on GP13 and extend the pattern to cycle through all three LEDs in sequence.
2. **Variable Speed**: Connect a potentiometer to GP26 and use its ADC reading to set the blink delay dynamically.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Both LEDs ON at same time | Logic inverted | Verify that `led1.value(1)` and `led2.value(0)` are written together in the same half-cycle block. |
| One LED never lights | Loose connection | Check that both Anodes are connected to their respective GP pins with the resistor in series. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**. No compilation step is needed — click Run and the code executes immediately.

## Related Projects
- [01 - Pico LED Blink](01-pico-blink.md)
- [09 - Pico Traffic Light](09-pico-traffic-light.md)
