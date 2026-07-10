# 13 - Pico Button Counter

Count and display the number of times a push button has been pressed, printing the running total to the serial terminal.

## Goal
Learn how to detect rising-edge button transitions (not just held states) using a state-change detection pattern in MicroPython, and maintain a running counter variable.

## What You Will Build
A press counter:
- **Push Button (GP14)**: Each press increments a counter displayed in the serial terminal.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button | Terminal 1 | GP14 | Blue | Signal input — reads LOW when button is pressed |
| Push Button | Terminal 2 | GND | Black | Button shorts GP14 to GND when pressed |

> **Wiring tip:** The internal `PULL_UP` resistor keeps GP14 at 3.3 V (HIGH) when the button is open. When the button is pressed, it connects GP14 directly to GND (LOW). No external resistor is required on either terminal.
>
> **Edge detection:** This project counts only the moment the button transitions from HIGH to LOW (press event), NOT how long it is held. Holding the button down counts as only one press.

## Code
```python
from machine import Pin
import utime

button = Pin(14, Pin.IN, Pin.PULL_UP)

count    = 0
prev_val = 1    # Start assuming button is not pressed (HIGH = 1)

print("Button counter ready. Press the button!")

while True:
    curr_val = button.value()

    # Detect falling edge: HIGH → LOW (button just pressed)
    if prev_val == 1 and curr_val == 0:
        count += 1
        print("Press #" + str(count))

    prev_val = curr_val
    utime.sleep_ms(50)    # 50 ms polling / debounce delay
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and a **Push Button** onto the canvas.
2. Connect Button Terminal 1 to **GP14** and Terminal 2 to **GND**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the button on the canvas repeatedly and watch the count increase in the terminal.

## Expected Output
```
Button counter ready. Press the button!
Press #1
Press #2
Press #3
```

## Expected Canvas Behavior
- Each click of the canvas button adds one to the counter and prints the updated count to the terminal.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `prev_val = 1` | Stores the button state from the previous loop iteration — starts at 1 (unpressed). |
| `curr_val = button.value()` | Reads the current button state in this loop iteration. |
| `prev_val == 1 and curr_val == 0` | Detects a falling edge: the button just transitioned from HIGH (unpressed) to LOW (pressed). |
| `count += 1` | Increments the counter by 1 on each detected press event. |
| `prev_val = curr_val` | Updates the stored previous state for the next loop iteration. |

## Hardware & Safety Concept: Edge Detection vs Level Detection
Checking `button.value() == 0` in a simple `if` block would fire every loop iteration while the button is held — that is **level detection**. This project uses **edge detection**: it only fires once, at the exact moment the button transitions from unpressed to pressed. Edge detection is essential for counters, toggles, and event-driven systems where holding a button should not repeat the action.

## Try This! (Challenges)
1. **Reset Button**: Wire a second button to GP13 that resets the counter to zero when pressed.
2. **LED Flash**: Connect an LED to GP15 that briefly flashes with each counted press.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Counter jumps by 2–3 per press | Button bounce / debounce too short | Increase `sleep_ms(50)` to `sleep_ms(100)` to allow more contact settling time. |
| Counter never increments | Button not reaching GND | Check that Terminal 2 is firmly connected to GND, not floating. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**. The counter output appears in the MbedO serial terminal panel.

## Related Projects
- [03 - Pico Button LED](03-pico-button-led.md)
- [14 - Pico Dual Button LED](14-pico-dual-button-led.md)
