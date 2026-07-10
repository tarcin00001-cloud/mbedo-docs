# 09 - Pico Traffic Light

Simulate a three-phase traffic light sequence with Red, Yellow, and Green LEDs cycling automatically.

## Goal
Learn how to control multiple GPIO output pins in a timed sequence using lists and loops in MicroPython, applying real-world traffic light timing logic.

## What You Will Build
A traffic light sequencer:
- **Red LED (GP13)**: Stop phase — ON for 3 seconds.
- **Yellow LED (GP14)**: Caution phase — ON for 1 second.
- **Green LED (GP15)**: Go phase — ON for 3 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Red LED | `led` | Yes | Yes |
| Yellow LED | `led` | Yes | Yes |
| Green LED | `led` | Yes | Yes |
| 330 Ω Resistors × 3 | `resistor` | Optional in MbedO | Yes — one per LED |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Red LED | Anode (+, longer leg) | GP13 | Red | Stop phase output |
| Red LED | Cathode (−, shorter leg) | GND | Black | Shared ground rail |
| 330 Ω Resistor (Red) | Either leg | In series between GP13 and Red LED Anode | — | Current limiter |
| Yellow LED | Anode (+, longer leg) | GP14 | Yellow | Caution phase output |
| Yellow LED | Cathode (−, shorter leg) | GND | Black | Shared ground rail |
| 330 Ω Resistor (Yellow) | Either leg | In series between GP14 and Yellow LED Anode | — | Current limiter |
| Green LED | Anode (+, longer leg) | GP15 | Green | Go phase output |
| Green LED | Cathode (−, shorter leg) | GND | Black | Shared ground rail |
| 330 Ω Resistor (Green) | Either leg | In series between GP15 and Green LED Anode | — | Current limiter |

> **Wiring tip:** All three LED Cathodes and all three Resistors share the same GND rail on the breadboard. Only one GND pin on the Pico is needed — connect the rail to any GND pin.

## Code
```python
from machine import Pin
import utime

red    = Pin(13, Pin.OUT)
yellow = Pin(14, Pin.OUT)
green  = Pin(15, Pin.OUT)

def all_off():
    red.value(0)
    yellow.value(0)
    green.value(0)

# Traffic light phases: (pin, duration_seconds)
phases = [
    (red,    3),    # Stop
    (yellow, 1),    # Caution
    (green,  3),    # Go
    (yellow, 1),    # Caution (before red)
]

while True:
    for pin, duration in phases:
        all_off()
        pin.value(1)
        utime.sleep(duration)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **three LEDs** (Red, Yellow, Green) onto the canvas.
2. Connect Red LED Anode to **GP13**, Yellow to **GP14**, Green to **GP15**; connect all Cathodes to **GND**.
3. Paste the code, select **MicroPython** mode, and click **Run**.

## Expected Output
The traffic light cycles: Red (3 s) → Yellow (1 s) → Green (3 s) → Yellow (1 s) → Red (3 s) → …

## Expected Canvas Behavior
- Only one LED is ON at any time. The sequence advances automatically through the four phases.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `all_off()` | Turns all three LEDs OFF before activating the next phase, preventing two LEDs from overlapping. |
| `phases` list | Stores each (LED pin, duration) pair — makes it easy to adjust timing without changing the loop logic. |
| `for pin, duration in phases` | Iterates through each phase, unpacking the pin and duration for that step. |
| `pin.value(1)` | Turns ON only the LED for the current phase. |

## Hardware & Safety Concept: Traffic Light Safety Interlock
Real traffic light controllers include a hardware safety interlock that physically prevents red and green signals from being ON simultaneously — even if software malfunctions. The `all_off()` call before each phase transition in this code replicates this safety requirement: it guarantees no two conflicting signals are ever active at the same time.

## Try This! (Challenges)
1. **Pedestrian Button**: Wire a button to GP2 and add a pedestrian crossing request — hold green until the button is pressed, then start the red phase.
2. **Night Mode**: Add a second button to toggle the yellow LED into a slow-flash night mode only.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Two LEDs ON simultaneously | `all_off()` not called before activation | Verify `all_off()` is the first call inside each phase loop iteration. |
| Wrong colour order | Pins swapped | Re-check that Red→GP13, Yellow→GP14, Green→GP15 match your wiring. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [01 - Pico LED Blink](01-pico-blink.md)
- [02 - Pico Double Blink](02-pico-double-blink.md)
