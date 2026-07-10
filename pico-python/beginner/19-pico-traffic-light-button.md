# 19 - Pico Traffic Light Button

Build a pedestrian-crossing traffic light sequencer that interrupts the normal green loop when a walk request button is pressed.

## Goal
Learn how to implement interruptible state loops using conditional input polling, transitioning from free-running sequences to event-driven processes in MicroPython.

## What You Will Build
An interactive pedestrian crossing:
- **Red LED (GP13)**: Stop phase (normally OFF).
- **Yellow LED (GP14)**: Warning transition (normally OFF).
- **Green LED (GP15)**: Go phase (normally ON).
- **Pedestrian Button (GP12)**: Request walk phase. When pressed, the sequence triggers: Green OFF → Yellow ON (1s) → Red ON (3s) → Green ON.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Red, Yellow, Green LEDs | `led` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| 330 Ω Resistors × 3 | `resistor` | Optional in MbedO | Yes — one per LED |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Red LED | Anode (+, longer leg) | GP13 | Red | Stop signal |
| Yellow LED | Anode (+, longer leg) | GP14 | Yellow | Warning signal |
| Green LED | Anode (+, longer leg) | GP15 | Green | Go signal (default ON) |
| Push Button | Terminal 1 | GP12 | Blue | Pedestrian request input |
| Push Button | Terminal 2 | GND | Black | Shorts GP12 to GND on press |
| All Cathodes | Cathode (−) | GND | Black | Shared return path to ground |
| 330 Ω Resistors | Either leg | In series with LED Anodes | — | Current limiters |

> **Wiring tip:** The pedestrian button uses internal pull-ups on GP12. The three LEDs share a common ground line. By default, the Green LED is ON, and the system waits for the button to go LOW.

## Code
```python
from machine import Pin
import utime

red    = Pin(13, Pin.OUT)
yellow = Pin(14, Pin.OUT)
green  = Pin(15, Pin.OUT)
button = Pin(12, Pin.IN, Pin.PULL_UP)

def set_lights(r, y, g):
    red.value(r)
    yellow.value(y)
    green.value(g)

# Start state: Green ON
set_lights(0, 0, 1)

while True:
    # Poll button state
    if button.value() == 0:
        print("Pedestrian request detected!")
        
        # 1. Transition to caution (Yellow ON)
        set_lights(0, 1, 0)
        utime.sleep(1)
        
        # 2. Stop traffic (Red ON)
        set_lights(1, 0, 0)
        utime.sleep(3)
        
        # 3. Warning before Green (Red + Yellow ON)
        set_lights(1, 1, 0)
        utime.sleep(1)
        
        # 4. Return to Green
        set_lights(0, 0, 1)
        
        # Cool-down period to prevent immediate re-trigger
        utime.sleep(5)
        print("Traffic light ready for new requests.")
        
    utime.sleep_ms(50) # Polling cycle
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **three LEDs** (Red, Yellow, Green), and **Push Button** onto the canvas.
2. Connect Red to **GP13**, Yellow to **GP14**, Green to **GP15**, and Button to **GP12**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the button component on the canvas and watch the sequence trigger.

## Expected Output
```
Pedestrian request detected!
Traffic light ready for new requests.
```

## Expected Canvas Behavior
- The system sits on Green. Pressing the button causes Green to turn OFF, Yellow to turn ON, then Red to turn ON, and finally transitions back to Green.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `button.value() == 0` | Detects a pedestrian button press (active LOW). |
| `set_lights(0, 1, 0)` | Helper function that sets Red OFF, Yellow ON, Green OFF. |
| `utime.sleep(5)` | Standard cooldown lockout period, preventing pedestrians from repeatedly blocking vehicle traffic. |

## Hardware & Safety Concept: Cooldown Lockouts
Pedestrian safety crossings include cooldown lockout periods (typically 30 to 90 seconds). This prevents pedestrians from pressing the button repeatedly to block vehicle flow. The microcontroller ignores new button presses during the cooldown period.

## Try This! (Challenges)
1. **Audio Beeper**: Add a buzzer on GP11 that beeps slowly during the Red phase to guide visually impaired pedestrians.
2. **Double Button Confirmation**: Require a second button press on GP10 to confirm the request, simulating dual-panel crossings.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Button does not trigger crossing | Pin mapping mismatch | Verify the button is connected to GP12, and the internal pull-up is enabled. |
| Lights change instantly without delays | Delays not set | Ensure the `sleep` statements are included in the execution path. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [09 - Pico Traffic Light](09-pico-traffic-light.md)
- [14 - Pico Dual Button LED](14-pico-dual-button-led.md)
