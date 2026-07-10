# 167 - Pico Liquid Filling Speed

Build a beverage bottling control station that controls liquid flow rates dynamically based on container depth.

## Goal
Learn how to monitor analog level sensors, check digital flow switches, and control the opening of liquid dispensing valves using MicroPython.

## What You Will Build
An adjustable liquid bottling system:
- **Water Level Sensor (GP26)**: Measures liquid height in the bottle.
- **Flow Switch / Start Button (GP16)**: Starts the filling process.
- **Dispensing Valve Relay (GP10)**: Actuates a 5V solenoid water valve.
- **16x2 I2C LCD (GP4, GP5)**: Displays filling progress, active flow rates, and valve state updates.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Water Level Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| Push Button | `button` | Yes | Yes (or flow switch) |
| Relay Module | `relay` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Water Sensor | Signal OUT | GP26 | Yellow | Container depth analog input |
| Water Sensor | VCC / GND | 3.3V / GND | Red / Black | Power lines |
| Start Button | Terminal 1 | GP16 | Green | Start filling trigger |
| Start Button | Terminal 2 | GND | Black | Ground return |
| Relay Module | IN | GP10 | Orange | Solenoid valve control |
| Relay Module | VCC / GND | 5V / GND | Red / Black | Relay power |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The water level sensor is read by ADC GP26. The Start Button is wired to GP16 and configured with an internal pull-up. The Solenoid Valve Relay is driven by GP10. The LCD uses shared I2C pins GP4/GP5.

## Code
```python
from machine import Pin, ADC, I2C
import utime
from machine_lcd import I2cLcd

water_adc = ADC(26)
start_btn = Pin(16, Pin.IN, Pin.PULL_UP)
relay = Pin(10, Pin.OUT)
relay.value(0)  # Close valve initially

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Depth limits (16-bit ADC)
EMPTY_LEVEL = 8000
FULL_LEVEL  = 48000

filling_active = False

def reset_dispenser_state():
    global filling_active
    filling_active = False
    relay.value(0)
    lcd.clear()
    lcd.putstr("Bottling Station\nPress Start...  ")

reset_dispenser_state()
print("Bottling station monitor active.")

while True:
    level = water_adc.read_u16()
    start_pressed = (start_btn.value() == 0)
    
    # 1. Detect start button press
    if start_pressed and not filling_active:
        if level < EMPTY_LEVEL:
            filling_active = True
            relay.value(1)  # Open dispensing valve
            lcd.clear()
            lcd.putstr("Bottling Active")
        else:
            # Bottle not empty or missing
            lcd.clear()
            lcd.putstr("Insert Empty\nBottle & Try...")
            utime.sleep(1.5)
            reset_dispenser_state()
            
    # 2. Filling progress calculation
    if filling_active:
        progress = (level - EMPTY_LEVEL) * 100 // (FULL_LEVEL - EMPTY_LEVEL)
        progress = max(0, min(100, progress))
        
        lcd.move_to(0, 1)
        lcd.putstr("Filled: {:3d}%".format(progress))
        
        # Close valve when full
        if level >= FULL_LEVEL:
            filling_active = False
            relay.value(0)  # Close valve
            lcd.clear()
            lcd.putstr("Filling Done!\nRemove Bottle")
            utime.sleep(3.0)
            reset_dispenser_state()
            
    utime.sleep_ms(100)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Water Level Sensor** (represented by potentiometer), **Push Button**, **Relay**, and **I2C LCD** onto the canvas.
2. Connect Water Sensor to **GP26**, Button to **GP16**, Relay to **GP10**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the water sensor to empty (< 8000), click the Start button to open the relay, then slide the sensor to full (> 48000) to close it.

## Expected Output
Terminal:
```
Bottling station monitor active.
```

## Expected Canvas Behavior
* Startup: LCD reads `Press Start...`. Relay is OFF.
* Press Start (empty bottle): Relay turns ON (Valve opens). LCD reads `Filled: 0%`.
* Slide depth sensor up: LCD percentage rises (e.g. `Filled: 60%`).
* Depth reaches maximum (> 48000): Relay turns OFF (Valve closes). LCD reads `Filling Done!`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `progress = (level - EMPTY_LEVEL) * 100 // (FULL_LEVEL - EMPTY_LEVEL)` | Calculates the container filling percentage ratio from the analog readings using integer division. |
| `relay.value(0)` | Closes the water solenoid valve by setting the control pin low. |

## Hardware & Safety Concept: Dispenser Overflow Prevention
Liquid filling stations must prevent spills and overflows. If the water level sensor fails or gets dirty, it may keep reporting empty, causing the valve to stay open and overflow the bottle. To prevent this, smart systems use an **auto-shutoff timeout** (e.g. closing the valve if filling takes longer than 10 seconds) or add secondary mechanical overflow floats.

## Try This! (Challenges)
1. **Critical Overflow Alarm**: Sound a buzzer on GP14 if the water level exceeds a maximum limit of 56000.
2. **Emergency Stop**: Wire a second button on GP15 that closes the valve immediately and cancels the filling sequence.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Valve closes immediately after opening | Depth sensor calibration issue | Ensure the empty container sensor reading is below `EMPTY_LEVEL` (8000) before clicking the start trigger. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [130 - Pico Water Level Station LCD Buzzer](130-pico-water-level-station-lcd-buzzer.md)
- [146 - Pico Soil Moisture Auto Irrigator OLED](146-pico-soil-moisture-auto-irrigator-oled.md)
