# 30 - Pico Light Level Meter LEDs

Display a visual light level scale (bar graph) across three LEDs based on ambient light levels.

## Goal
Learn how to read an analog LDR light sensor and use multi-stage threshold logic to drive three separate digital output channels in MicroPython.

## What You Will Build
A visual light level indicator:
- **LDR Sensor (GP26)**: Measures ambient light intensity.
- **LED 1 (GP13 - Red)**: Lights up when light is low.
- **LED 2 (GP14 - Yellow)**: Lights up when light is moderate.
- **LED 3 (GP15 - Green)**: Lights up when light is bright.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LDR Photoresistor | `ldr` | Yes | Yes |
| Red, Yellow, Green LEDs | `led` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | Optional in MbedO | Yes (voltage divider) |
| 330 Ω Resistors × 3 | `resistor` | Optional in MbedO | Yes — one per LED |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR | Pin 1 | 3.3V (3V3) | Red | Power line |
| LDR | Pin 2 | GP26 | Yellow | Analog input signal |
| 10 kΩ Resistor | Either leg | GP26 to GND | Black | Pull-down resistor |
| LED 1 (Red) | Anode (+, longer leg) | GP13 | Red | Low light indicator |
| LED 2 (Yellow) | Anode (+, longer leg) | GP14 | Yellow | Moderate light indicator |
| LED 3 (Green) | Anode (+, longer leg) | GP15 | Green | High light indicator |
| All Cathodes | Cathode (−) | GND | Black | Shared return path to ground |
| 330 Ω Resistors | Either leg | In series with LED Anodes | — | Limits LED current |

> **Wiring tip:** The LDR and 10 kΩ resistor form a voltage divider connected to GP26. The three LED anodes connect to GP13, GP14, and GP15 through 330 Ω resistors. All cathodes share a common ground line.

## Code
```python
from machine import Pin, ADC
import utime

ldr = ADC(26) # GP26 = ADC channel 0

led_red    = Pin(13, Pin.OUT)
led_yellow = Pin(14, Pin.OUT)
led_green  = Pin(15, Pin.OUT)

# Threshold limits (0 - 65535 raw ADC range)
LIMIT_LOW  = 15000 # Under this: Red ON
LIMIT_MID  = 30000 # Under this: Red + Yellow ON
                   # Above this: Red + Yellow + Green ON

def set_leds(r, y, g):
    led_red.value(r)
    led_yellow.value(y)
    led_green.value(g)

while True:
    val = ldr.read_u16()
    print("Light reading:", val)
    
    # Threshold logic
    if val < LIMIT_LOW:
        set_leds(1, 0, 0) # Only Red ON (Low light)
    elif val < LIMIT_MID:
        set_leds(1, 1, 0) # Red and Yellow ON (Moderate light)
    else:
        set_leds(1, 1, 1) # Red, Yellow, and Green ON (Bright light)
        
    utime.sleep_ms(200) # Polling update delay
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **LDR**, and **three LEDs** (Red, Yellow, Green) onto the canvas.
2. Connect LDR Pin 1 to **3.3V** and Pin 2 to **GP26**.
3. Connect Red LED to **GP13**, Yellow to **GP14**, Green to **GP15**. Connect all grounds.
4. Paste the code, select **MicroPython** mode, and click **Run**.
5. Adjust the LDR slider on the canvas and watch the LEDs light up.

## Expected Output
```
Light reading: 8000
Light reading: 22000
Light reading: 45000
```

## Expected Canvas Behavior
- The LEDs light up in sequence as you move the LDR slider up, representing a volume bar graph.
- Sliding the LDR down turns the LEDs OFF in reverse sequence.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `val < LIMIT_LOW` | Checks if the ambient light level is below the low threshold limit. |
| `set_leds(1, 1, 0)` | Turns the Red and Yellow LEDs ON, and the Green LED OFF. |

## Hardware & Safety Concept: Bar Graph Displays
LED bar graphs (such as VU audio level meters or battery capacity indicators) provide clear visual readouts of analog values. Using multiple LEDs in sequence allows users to check levels at a glance without reading numbers on a screen.

## Try This! (Challenges)
1. **Level Chaser**: Change the logic so only ONE LED is ON at a time to represent the light level, rather than filling the bar.
2. **Reverse Indicator**: Configure the LEDs to act as a darkness indicator bar graph (brighter room → fewer LEDs lit).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LEDs stay ON all the time | Thresholds set too low | Check the ambient light levels in the terminal and increase `LIMIT_LOW` and `LIMIT_MID` in your code. |
| LEDs do not light up | Wiring error on cathodes | Ensure the ground connection (cathode) for all three LEDs is firmly connected to GND. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [02 - Pico Double Blink](02-pico-double-blink.md)
- [09 - Pico Traffic Light](09-pico-traffic-light.md)
- [10 - Pico LDR Light Sensor Serial](10-pico-ldr-serial.md)
- [22 - Pico Night Light LDR](22-pico-night-light-ldr.md)
