# 118 - Pico Dual Relay Timer Controller LCD

Build a two-channel timed relay controller that activates and deactivates two independent relay outputs on configurable timer schedules, and displays countdown timers on an I2C LCD.

## Goal
Learn how to implement independent non-blocking software timers using `utime.ticks_ms()`, control two relay outputs on separate schedules, and display countdown values on an I2C character LCD in MicroPython.

## What You Will Build
A dual-channel relay timer:
- **Relay 1 (GP14)**: Activates for 5 seconds, then waits 5 seconds before repeating.
- **Relay 2 (GP15)**: Activates for 3 seconds, then waits 7 seconds before repeating.
- **Button (GP13)**: Resets both timers immediately.
- **I2C 16x2 LCD (GP4, GP5)**: Displays the ON/OFF state and remaining seconds for each relay.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Relay Module × 2 | `relay` | Yes | Yes |
| Tactile Push Button | `button` | Yes (reset switch) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Relay 1 | IN | GP14 | Orange | Channel 1 relay control |
| Relay 2 | IN | GP15 | Yellow | Channel 2 relay control |
| Reset Button | Terminal 1 | GP13 | White | Reset input (reads LOW on press) |
| Reset Button | Terminal 2 | GND | Black | Shorts GP13 to GND |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect Relay 1 to GP14 and Relay 2 to GP15. Connect the reset button to GP13. Connect the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime
from machine_lcd import I2cLcd

relay1 = Pin(14, Pin.OUT)
relay2 = Pin(15, Pin.OUT)
btn    = Pin(13, Pin.IN, Pin.PULL_UP)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Timer configuration (all values in milliseconds)
R1_ON_MS  = 5000  # Relay 1 ON for 5 seconds
R1_OFF_MS = 5000  # Relay 1 OFF for 5 seconds
R2_ON_MS  = 3000  # Relay 2 ON for 3 seconds
R2_OFF_MS = 7000  # Relay 2 OFF for 7 seconds

def reset_timers():
    relay1.value(1) # Relay 1 starts ON
    relay2.value(1) # Relay 2 starts ON
    return (utime.ticks_ms(), True, utime.ticks_ms(), True)

relay1.value(0)
relay2.value(0)

t1_start, r1_on, t2_start, r2_on = reset_timers()

lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("Dual Timer Panel")
lcd.move_to(0, 1)
lcd.putstr("Initializing...")
utime.sleep(1)

print("Dual relay timer active.")

while True:
    now = utime.ticks_ms()
    
    # --- Relay 1 timer logic ---
    elapsed1 = utime.ticks_diff(now, t1_start)
    limit1    = R1_ON_MS if r1_on else R1_OFF_MS
    remain1_s = max(0, (limit1 - elapsed1) // 1000)
    
    if elapsed1 >= limit1:
        r1_on   = not r1_on
        t1_start = now
        relay1.value(1 if r1_on else 0)
        
    # --- Relay 2 timer logic ---
    elapsed2 = utime.ticks_diff(now, t2_start)
    limit2    = R2_ON_MS if r2_on else R2_OFF_MS
    remain2_s = max(0, (limit2 - elapsed2) // 1000)
    
    if elapsed2 >= limit2:
        r2_on   = not r2_on
        t2_start = now
        relay2.value(1 if r2_on else 0)
        
    # --- Reset button ---
    if btn.value() == 0:
        t1_start, r1_on, t2_start, r2_on = reset_timers()
        print(">> Timers reset.")
        utime.sleep_ms(300) # Debounce
        
    # --- Update LCD ---
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("R1:{} {:2d}s R2:{} {:2d}s".format(
        "ON " if r1_on else "OFF", remain1_s,
        "ON " if r2_on else "OFF", remain2_s))
    lcd.move_to(0, 1)
    lcd.putstr("Press BTN=Reset")
    
    utime.sleep_ms(250) # Update rate
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two Relays**, **Push Button**, and **I2C LCD** onto the canvas.
2. Connect Relay 1 to **GP14**, Relay 2 to **GP15**, Button to **GP13**, and LCD to **GP4/GP5**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Observe the two relays toggling on independent schedules and the LCD showing their countdown timers.

## Expected Output
```
Dual relay timer active.
```
(On screen: "R1:ON  5s R2:ON  3s" updating each second.)

## Expected Canvas Behavior
- The two relay components on the canvas toggle independently on their configured ON/OFF schedules. The LCD countdown timers count down to zero and then reset.
- Clicking the Button resets both timers immediately.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `utime.ticks_diff(now, t1_start)` | Calculates elapsed time without integer overflow, using the MicroPython monotonic clock. |
| `r1_on = not r1_on` | Toggles the relay state flag when the current phase timer expires. |

## Hardware & Safety Concept: Non-Blocking Timer Patterns
Traditional `utime.sleep()` calls block the CPU. For multi-channel control, the code uses `utime.ticks_ms()` timestamps to measure elapsed time without blocking, allowing both relay timers to run independently in the same main loop.

## Try This! (Challenges)
1. **Configurable Timers**: Add two potentiometers on GP26/GP27 to adjust the ON duration for each relay dynamically.
2. **Manual Override**: Add a second button on GP12 that manually toggles Relay 1 state regardless of the timer.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Both relays switch together | Timer intervals identical | Verify that R1 and R2 have different ON and OFF durations configured. |
| Reset button not responding | Debounce delay issue | Ensure the 300 ms debounce sleep in the reset block is present. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [62 - Pico Relay Button](62-pico-relay-button.md)
- [92 - Pico Relay Button LCD](92-pico-relay-button-lcd.md)
