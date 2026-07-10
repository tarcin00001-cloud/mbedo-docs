# 62 - Pico Relay Button

Toggle a high-load power relay ON and OFF using a momentary push button.

## Goal
Learn how to implement a software toggle switch (latching state) that changes the state of a digital relay output pin each time a button is pressed in MicroPython.

## What You Will Build
A latching relay controller:
- **Push Button (GP14)**: Toggles the state of the relay (active LOW).
- **Relay Module (GP15)**: Switches high-load power circuits (represented by a simulated bulb or motor) ON and OFF on alternate button presses.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button | Terminal 1 | GP14 | Blue | Button signal input (reads LOW on press) |
| Push Button | Terminal 2 | GND | Black | Shorts GP14 to GND on press |
| Relay Module | VCC | 5V (VBUS) or 3.3V | Red | Power line |
| Relay Module | GND | GND | Black | Ground reference |
| Relay Module | IN | GP15 | Orange | Relay control signal (HIGH on activate) |

> **Wiring tip:** The button uses the internal pull-up on GP14. Connect the relay input pin to GP15. All ground connections are shared.

## Code
```python
from machine import Pin
import utime

button = Pin(14, Pin.IN, Pin.PULL_UP)
relay  = Pin(15, Pin.OUT)

relay_state = False
relay.value(0) # Start with relay OFF

prev_val = 1
button.value()

print("Relay toggle console active. Press the button!")

while True:
    curr_val = button.value()
    
    # Detect falling edge: HIGH -> LOW (button pressed)
    if prev_val == 1 and curr_val == 0:
        relay_state = not relay_state # Toggle state
        relay.value(1 if relay_state else 0)
        
        print("Relay Toggled:", "ON" if relay_state else "OFF")
        utime.sleep_ms(200) # Lockout delay to debounce button contacts
        
    prev_val = curr_val
    utime.sleep_ms(20) # Polling update speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Push Button**, and **Relay** onto the canvas.
2. Connect Button to **GP14** and Relay to **GP15**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the button component on the canvas and observe the relay toggling.

## Expected Output
```
Relay toggle console active. Press the button!
Relay Toggled: ON
Relay Toggled: OFF
```

## Expected Canvas Behavior
- The relay component clicks and changes state (closes/opens the contact) on alternate clicks of the button.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `relay_state = not relay_state` | Inverts the boolean flag representing the relay state (toggles between `True` and `False`). |
| `relay.value(1 if relay_state else 0)` | Activates the relay output (drives GP15 HIGH) if the state is `True`, and deactivates it if `False`. |

## Hardware & Safety Concept: Galvanic Isolation
Relays provide **galvanic isolation** between low-voltage microcontrollers (like the 3.3V Pico) and high-voltage power circuits (like 240V mains). Because the control coil and the high-power contacts are completely separate electrically, high-voltage faults cannot travel back to damage the microcontroller or shock the user.

## Try This! (Challenges)
1. **Indicator LED**: Add an LED on GP13 that turns ON when the relay is active.
2. **Auto-Off Timer**: Modify the code so that the relay turns OFF automatically if left ON for more than 5 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay clicks multiple times per press | Button bounce issues | Increase the debounce delay (`sleep_ms(200)`) after the state changes. |
| Relay does not click | Insufficient coil current | Ensure the relay VCC is connected to the Pico's 5V VBUS pin to provide enough coil current. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [03 - Pico Button LED](03-pico-button-led.md)
- [13 - Pico Button Counter](13-pico-button-counter.md)
- [40 - Pico Alarm System Button Buzzer](40-pico-alarm-system-button-buzzer.md)
