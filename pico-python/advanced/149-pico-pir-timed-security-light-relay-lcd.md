# 149 - Pico PIR Timed Security Light Relay LCD

Build a passive-infrared security light controller that activates a relay-driven floodlight on motion detection, holds it on for a configurable duration, displays a countdown timer on an I2C LCD, and supports manual override via a button.

## Goal
Learn how to implement a timed output driven by a PIR sensor, use a non-blocking countdown timer to control relay hold duration, provide a manual override button for persistent on/off, and display the countdown and mode on an I2C LCD in MicroPython.

## What You Will Build
A timed PIR security light:
- **PIR Sensor (GP14)**: Triggers light activation on motion.
- **Relay — Floodlight (GP10)**: Drives the external light.
- **Button (GP13)**: Cycles through AUTO → FORCE ON → FORCE OFF modes.
- **I2C 16x2 LCD (GP4, GP5)**: Displays mode, relay state, and countdown remaining.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| PIR Motion Sensor | `button` | Yes (digital output) | Yes |
| Relay Module | `relay` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| PIR Sensor | VCC | 5V (VBUS) | Red | PIR power (5V) |
| PIR Sensor | GND | GND | Black | Ground reference |
| PIR Sensor | OUT | GP14 | Blue | HIGH = motion detected |
| Relay Module | IN | GP10 | Orange | Floodlight relay control |
| Mode Button | Terminal 1 | GP13 | White | Cycles operating modes |
| Mode Button | Terminal 2 | GND | Black | Button return |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect PIR OUT to GP14, relay to GP10, mode button to GP13 (with PULL_UP), and LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime
from machine_lcd import I2cLcd

pir      = Pin(14, Pin.IN, Pin.PULL_DOWN)
relay    = Pin(10, Pin.OUT)
mode_btn = Pin(13, Pin.IN, Pin.PULL_UP)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

relay.value(0)

# Operating modes
MODE_AUTO      = 0   # PIR-controlled with timer
MODE_FORCE_ON  = 1   # Always ON regardless of PIR
MODE_FORCE_OFF = 2   # Always OFF regardless of PIR

MODE_NAMES = ["AUTO    ", "FORCE ON", "FORCE OF"]

mode       = MODE_AUTO
hold_secs  = 30         # Auto-mode hold duration in seconds
countdown  = 0          # Remaining hold seconds
last_btn   = 0
last_tick  = utime.ticks_ms()

lcd.clear(); lcd.putstr("PIR Sec Light"); utime.sleep(1)
print("PIR timed security light active.")

while True:
    now = utime.ticks_ms()

    # --- Mode button ---
    if mode_btn.value() == 0 and utime.ticks_diff(now, last_btn) > 500:
        last_btn = now
        mode = (mode + 1) % 3
        if mode == MODE_FORCE_OFF:
            relay.value(0); countdown = 0
        elif mode == MODE_FORCE_ON:
            relay.value(1); countdown = 0
        print("Mode:", MODE_NAMES[mode])

    # --- Auto mode PIR logic ---
    if mode == MODE_AUTO:
        if pir.value() == 1:
            relay.value(1)
            countdown = hold_secs
            print("Motion! Hold for {}s.".format(hold_secs))

        # Tick countdown every second
        if utime.ticks_diff(now, last_tick) >= 1000:
            last_tick = now
            if countdown > 0:
                countdown -= 1
                if countdown == 0:
                    relay.value(0)
                    print("Timer expired. Light OFF.")

    # --- Force modes ---
    elif mode == MODE_FORCE_ON:
        relay.value(1)
    elif mode == MODE_FORCE_OFF:
        relay.value(0)

    # --- LCD update ---
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("Mode: " + MODE_NAMES[mode])
    lcd.move_to(0, 1)
    if mode == MODE_AUTO:
        if relay.value():
            lcd.putstr("ON  Hold: {:3d}s".format(countdown))
        else:
            lcd.putstr("OFF  Standby   ")
    else:
        lcd.putstr("Light: " + ("ON " if relay.value() else "OFF"))

    utime.sleep_ms(100)
```

## What to Click in MbedO
1. Drag **Pico**, **PIR Button**, **Relay**, **Mode Button**, and **I2C LCD** onto the canvas.
2. Connect PIR to **GP14**, Relay to **GP10**, Mode Button to **GP13**, LCD to **GP4/GP5**. Connect power and GND.
3. Paste code, click **Run**. Click the PIR button to trigger the light. Observe the 30-second countdown on LCD. Click Mode Button to cycle through AUTO → FORCE ON → FORCE OFF.

## Expected Output
```
PIR timed security light active.
Motion! Hold for 30s.
Timer expired. Light OFF.
Mode: FORCE ON
Mode: FORCE OF
Mode: AUTO
```

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `countdown = hold_secs` on PIR trigger | Re-arms the hold timer every time new motion is detected, extending the on-time if motion continues. |
| `utime.ticks_diff(now, last_tick) >= 1000` | Non-blocking 1-second tick: decrements countdown each second without using `sleep()`. |

## Hardware & Safety Concept: PIR Re-Trigger and Hold Timers
Commercial security lights use a fixed or adjustable hold time (typically 20–300 seconds) plus optional retrigger: if motion continues, the timer restarts, keeping the light on. This prevents the light from flicking off while someone is present, while still turning off automatically when the area is empty.

## Try This! (Challenges)
1. **Adjustable Hold Time**: Add buttons to increase or decrease `hold_secs` in 10-second steps and display the setting on the LCD.
2. **Dusk Sensor**: Connect an LDR on GP26 and only enable AUTO mode when ambient light is below a threshold, preventing daytime operation.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Light activates without motion | PIR in re-trigger mode | Set PIR jumper to L (single-trigger) mode to prevent continuous HIGH during motion. |
| Countdown never reaches 0 | Tick timer comparison error | Ensure `last_tick` is updated inside the `>= 1000` block after the decrement. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [113 - Pico Auto Night Light LDR PIR LCD](../intermediate/113-pico-auto-night-light-ldr-pir-lcd.md)
- [129 - Pico Smart Doorbell PIR Buzzer LCD](129-pico-smart-doorbell-pir-buzzer-lcd.md)
