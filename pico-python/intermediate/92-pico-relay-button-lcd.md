# 92 - Pico Relay Button LCD

Build a latching power control panel that toggles a high-load relay ON and OFF using a momentary button, and displays the active power state on an I2C LCD.

## Goal
Learn how to implement a software toggle switch (latching state) that changes the state of a digital relay output pin, and update status text on an I2C character LCD in MicroPython.

## What You Will Build
A latching power switch console:
- **Push Button (GP14)**: Toggles the state of the relay (active LOW).
- **Relay Module (GP15)**: Switches high-load power circuits (represented by a simulated bulb or motor) ON and OFF on alternate button presses.
- **I2C 16x2 LCD (GP4, GP5)**: Displays the active state of the load ("LOAD STATE: ON" or "LOAD STATE: OFF").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button | Terminal 1 | GP14 | Blue | Button signal input (reads LOW on press) |
| Push Button | Terminal 2 | GND | Black | Shorts GP14 to GND on press |
| Relay Module | IN | GP15 | Orange | Relay control signal (HIGH on activate) |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The button connects between GP14 and GND. Connect the relay input pin to GP15, and the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime
from machine_lcd import I2cLcd

button = Pin(14, Pin.IN, Pin.PULL_UP)
relay  = Pin(15, Pin.OUT)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

relay_state = False
relay.value(0) # Start with relay OFF

prev_val = 1
button.value()

# Initial LCD update
lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("Power Console")
lcd.move_to(0, 1)
lcd.putstr("Load State: OFF")

print("Relay toggle console armed. Press the button!")

while True:
    curr_val = button.value()
    
    # Detect falling edge: HIGH -> LOW (button pressed)
    if prev_val == 1 and curr_val == 0:
        relay_state = not relay_state # Toggle state
        relay.value(1 if relay_state else 0)
        
        # Update LCD Display
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("Switch Toggled!")
        lcd.move_to(0, 1)
        lcd.putstr("Load: " + ("ON " if relay_state else "OFF"))
        
        print("Relay Toggled:", "ON" if relay_state else "OFF")
        utime.sleep_ms(200) # Lockout delay to debounce button contacts
        
    prev_val = curr_val
    utime.sleep_ms(20) # Polling update speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Push Button**, **Relay**, and **I2C LCD** onto the canvas.
2. Connect Button to **GP14**, Relay to **GP15**, and LCD to **GP4/GP5**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the button component on the canvas and observe the relay and LCD display update.

## Expected Output
```
Relay toggle console armed. Press the button!
Relay Toggled: ON
Relay Toggled: OFF
```
(On screen: "Load: ON" or "Load: OFF" on line 2, updating on button clicks.)

## Expected Canvas Behavior
- The relay component clicks and changes state, and the LCD screen updates to show the load state on alternate clicks of the button.

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
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [62 - Pico Relay Button](62-pico-relay-button.md)
- [92 - Pico Relay Button LCD](92-pico-relay-button-lcd.md)
