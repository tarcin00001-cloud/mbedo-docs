# 119 - Pico Traffic Light Controller LCD

Build a programmable traffic light controller that cycles through a red/amber/green LED sequence with configurable phase durations, and displays a countdown timer and active phase on an I2C LCD.

## Goal
Learn how to drive a traffic light LED sequence using state machine logic with configurable phase durations, implement non-blocking countdown timers, and display the active phase on an I2C character LCD in MicroPython.

## What You Will Build
A traffic light controller:
- **Red LED (GP12)**: ON during the STOP phase.
- **Amber LED (GP11)**: ON during the CAUTION phase.
- **Green LED (GP10)**: ON during the GO phase.
- **I2C 16x2 LCD (GP4, GP5)**: Displays the current phase label and seconds remaining.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Red LED | `led` | Yes | Yes |
| Amber LED | `led` | Yes | Yes |
| Green LED | `led` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 330 Ω Resistors × 3 | `resistor` | Optional in MbedO | Yes (one per LED) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Green LED | Anode (+, via 330 Ω) | GP10 | Green | GO phase indicator |
| Amber LED | Anode (+, via 330 Ω) | GP11 | Yellow | CAUTION phase indicator |
| Red LED | Anode (+, via 330 Ω) | GP12 | Red | STOP phase indicator |
| All LEDs | Cathode (−) | GND | Black | Shared return path to ground |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect each LED through a 330 Ω resistor to its control pin. Connect the LCD to GP4/GP5. All cathodes and grounds are shared on the common ground rail.

## Code
```python
from machine import Pin, I2C
import utime
from machine_lcd import I2cLcd

led_green = Pin(10, Pin.OUT)
led_amber = Pin(11, Pin.OUT)
led_red   = Pin(12, Pin.OUT)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Phase configuration: (label, LED pins: G/A/R, duration seconds)
PHASES = [
    ("GO     ", (1, 0, 0), 10),    # Green ON for 10 seconds
    ("CAUTION", (0, 1, 0), 3),     # Amber ON for 3 seconds
    ("STOP   ", (0, 0, 1), 8),     # Red ON for 8 seconds
]

def set_leds(green, amber, red):
    led_green.value(green)
    led_amber.value(amber)
    led_red.value(red)

set_leds(0, 0, 0)

lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("Traffic Light")
lcd.move_to(0, 1)
lcd.putstr("Starting...")
utime.sleep(1)

print("Traffic light controller active.")

phase_index = 2 # Start with STOP phase (safe state)

while True:
    label, (g, a, r), duration = PHASES[phase_index]
    
    # Activate LEDs for this phase
    set_leds(g, a, r)
    phase_start = utime.ticks_ms()
    
    print("Phase: {} | Duration: {} s".format(label.strip(), duration))
    
    # Non-blocking countdown loop
    while True:
        elapsed = utime.ticks_diff(utime.ticks_ms(), phase_start) // 1000
        remain  = max(0, duration - elapsed)
        
        # Update LCD each second
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("Phase: " + label)
        lcd.move_to(0, 1)
        lcd.putstr("Time:  {:2d} s".format(remain))
        
        if remain == 0:
            break
            
        utime.sleep_ms(200) # Update resolution
        
    # Advance to next phase
    phase_index = (phase_index + 1) % len(PHASES)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **three LEDs** (Green, Amber, Red), and **I2C LCD** onto the canvas.
2. Connect Green LED to **GP10**, Amber LED to **GP11**, Red LED to **GP12**, and LCD to **GP4/GP5**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Observe the LEDs cycling through the traffic light sequence while the LCD counts down each phase.

## Expected Output
```
Traffic light controller active.
Phase: STOP | Duration: 8 s
Phase: GO | Duration: 10 s
Phase: CAUTION | Duration: 3 s
```
(On screen: "Phase: GO     " and "Time:  10 s" counting down.)

## Expected Canvas Behavior
- The Red, Amber, and Green LED components on the canvas light up in sequence. The LCD displays the active phase label and a live countdown timer.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `PHASES = [...]` | Defines a list of state-machine phases, each with a label, LED states, and duration. |
| `phase_index = (phase_index + 1) % len(PHASES)` | Advances to the next phase and wraps around to zero when the last phase finishes. |

## Hardware & Safety Concept: Fail-Safe Traffic Light Initialization
Traffic light controllers always power on in the **all-red** state (safe state). This prevents green lights on conflicting approaches from appearing simultaneously during startup or after a power failure. The code starts at `phase_index = 2` (STOP) for the same reason.

## Try This! (Challenges)
1. **Manual Override Button**: Add a button on GP13 that forces an emergency all-red state when held down.
2. **Second Approach**: Add three more LEDs (GP2-GP4) for a second traffic direction that is anti-phased (green when first is red).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Multiple LEDs ON simultaneously | Missing `set_leds(0,0,0)` before phase change | Verify the tuple-based LED state correctly sets only one LED HIGH per phase. |
| Countdown stays at same second | `utime.sleep_ms(200)` too slow | The countdown resolution is 200 ms. If missed, the display may skip from 2 to 0. This is cosmetic. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [01 - Pico LED Blink](01-pico-led-blink.md)
- [08 - Pico Traffic Light](08-pico-traffic-light.md)
- [34 - Pico Traffic Light LCD](34-pico-traffic-light-lcd.md)
