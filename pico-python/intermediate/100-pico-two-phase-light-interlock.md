# 100 - Pico Two-Phase Light Interlock

Build an alternating safety warning beacon with two flashing LEDs that pulse in opposite phases.

## Goal
Learn how to control multiple output pins in opposite phases to create an alternating flashing pattern in MicroPython.

## What You Will Build
An alternating warning beacon:
- **LED 1 (GP14 - Red)**: ON when LED 2 is OFF.
- **LED 2 (GP15 - Red)**: ON when LED 1 is OFF.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Red LEDs × 2 | `led` | Yes | Yes |
| 330 Ω Resistors × 2 | `resistor` | Optional in MbedO | Yes — one per LED |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED 1 | Anode (+, longer leg) | GP14 | Red | Phase 1 warning signal |
| LED 1 | Cathode (−, shorter leg) | GND | Black | Shared return path to ground |
| 330 Ω Resistor 1 | Either leg | In series between GP14 and LED 1 Anode | — | Current limiter |
| LED 2 | Anode (+, longer leg) | GP15 | Orange | Phase 2 warning signal |
| LED 2 | Cathode (−, shorter leg) | GND | Black | Shared return path to ground |
| 330 Ω Resistor 2 | Either leg | In series between GP15 and LED 2 Anode | — | Current limiter |

> **Wiring tip:** Both LEDs share a common ground line. Connect LED 1 anode to GP14, and LED 2 anode to GP15. Place resistors in series with each anode.

## Code
```python
from machine import Pin
import utime

led1 = Pin(14, Pin.OUT)
led2 = Pin(15, Pin.OUT)

print("Warning beacon active.")

while True:
    # Phase 1: LED 1 ON, LED 2 OFF
    led1.value(1)
    led2.value(0)
    utime.sleep_ms(500)
    
    # Phase 2: LED 1 OFF, LED 2 ON
    led1.value(0)
    led2.value(1)
    utime.sleep_ms(500)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **two LEDs** onto the canvas.
2. Connect LED 1 to **GP14** and LED 2 to **GP15**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Observe the alternating flashing pattern.

## Expected Output
```
Warning beacon active.
```
(On canvas: LEDs alternate flashing every 500 ms.)

## Expected Canvas Behavior
- The two LED components on the canvas glow and dim alternately every half second.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `led1.value(1); led2.value(0)` | Activates Phase 1: turns LED 1 ON and LED 2 OFF. |
| `utime.sleep_ms(500)` | Holds the phase state for 500 milliseconds before transitioning. |

## Hardware & Safety Concept: Alternating Flashing Beacons
Alternating flashing beacons (e.g. railway crossing signals or industrial warning strobes) are designed to attract attention more effectively than a single blinking light. Toggling two sources in opposite phases ensures a constant level of light emission, which increases visibility in poor weather conditions.

## Try This! (Challenges)
1. **Speed Adjuster**: Connect a potentiometer on GP26 to dynamically adjust the flashing rate.
2. **Double Flash Pulse**: Implement a double-strobe pattern (two quick flashes on LED 1, then two quick flashes on LED 2).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Both LEDs flash together | Inverted logic states | Check your code to ensure one LED is written HIGH while the other is written LOW in each phase. |
| One LED does not flash | Loose connection | Check the anode and cathode connections for the LED that is not lighting up. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [01 - Pico LED Blink](01-pico-blink.md)
- [02 - Pico Double Blink](02-pico-double-blink.md)
- [09 - Pico Traffic Light](09-pico-traffic-light.md)
