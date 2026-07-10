# 32 - Pico Brightness Level Indicator

Display a visual rotation scale (bar graph) across three separate LEDs based on potentiometer adjustments.

## Goal
Learn how to read an analog potentiometer value (0–65535) and use multi-stage threshold logic to drive three separate digital output channels in MicroPython.

## What You Will Build
A rotary level indicator:
- **Potentiometer (GP26)**: Measures rotation angle.
- **LED 1 (GP13 - Red)**: Lights up when rotation exceeds 25%.
- **LED 2 (GP14 - Yellow)**: Lights up when rotation exceeds 50%.
- **LED 3 (GP15 - Green)**: Lights up when rotation exceeds 75%.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| Red, Yellow, Green LEDs | `led` | Yes | Yes |
| 330 Ω Resistors × 3 | `resistor` | Optional in MbedO | Yes — one per LED |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Left Pin (Pin 1) | GND | Black | Ground reference |
| Potentiometer | Wiper (Pin 2 - centre) | GP26 | Yellow | Analog input pin |
| Potentiometer | Right Pin (Pin 3) | 3.3V (3V3) | Red | High voltage reference |
| LED 1 (Red) | Anode (+, longer leg) | GP13 | Red | Low level indicator |
| LED 2 (Yellow) | Anode (+, longer leg) | GP14 | Yellow | Mid level indicator |
| LED 3 (Green) | Anode (+, longer leg) | GP15 | Green | High level indicator |
| All Cathodes | Cathode (−) | GND | Black | Shared return path to ground |
| 330 Ω Resistors | Either leg | In series with LED Anodes | — | Limits LED current |

> **Wiring tip:** Connect the potentiometer wiper pin to GP26. Connect the three LED anodes to GP13, GP14, and GP15. Connect all cathodes to ground.

## Code
```python
from machine import Pin, ADC
import utime

pot = ADC(26) # GP26 = ADC channel 0

led_red    = Pin(13, Pin.OUT)
led_yellow = Pin(14, Pin.OUT)
led_green  = Pin(15, Pin.OUT)

# Threshold limits (0 - 65535 raw ADC range)
LIMIT_LOW  = 16384 # 25% rotation
LIMIT_MID  = 32768 # 50% rotation
LIMIT_HIGH = 49152 # 75% rotation

def set_leds(r, y, g):
    led_red.value(r)
    led_yellow.value(y)
    led_green.value(g)

while True:
    val = pot.read_u16()
    print("Potentiometer value:", val)
    
    # Bar graph levels
    if val > LIMIT_HIGH:
        set_leds(1, 1, 1) # All LEDs ON (High level)
    elif val > LIMIT_MID:
        set_leds(1, 1, 0) # Red + Yellow ON (Mid level)
    elif val > LIMIT_LOW:
        set_leds(1, 0, 0) # Only Red ON (Low level)
    else:
        set_leds(0, 0, 0) # All OFF
        
    utime.sleep_ms(100) # Update rate
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Potentiometer**, and **three LEDs** (Red, Yellow, Green) onto the canvas.
2. Connect Potentiometer wiper to **GP26**.
3. Connect Red LED to **GP13**, Yellow to **GP14**, Green to **GP15**. Connect all grounds.
4. Paste the code, select **MicroPython** mode, and click **Run**.
5. Slide the potentiometer knob on the canvas and watch the LEDs light up.

## Expected Output
```
Potentiometer value: 8000
Potentiometer value: 22000
Potentiometer value: 54000
```

## Expected Canvas Behavior
- The LEDs light up in sequence (Red → Yellow → Green) as you slide the potentiometer from bottom to top.
- Sliding the potentiometer down turns the LEDs OFF in reverse sequence.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `val > LIMIT_HIGH` | Checks if the potentiometer wiper voltage exceeds the 75% threshold. |
| `set_leds(1, 1, 0)` | Turns the Red and Yellow LEDs ON, and the Green LED OFF. |

## Hardware & Safety Concept: Rotary Scale Readouts
Rotary scale displays are commonly used to show settings like volume levels, speed adjustments, or battery state-of-charge. Visual feedback helps users check adjustments quickly without needing to look at a text screen.

## Try This! (Challenges)
1. **Level Chaser**: Change the logic so only ONE LED is ON at a time to represent the potentiometer position (Red for low, Yellow for mid, Green for high).
2. **Reverse Indicator**: Configure the LEDs to act as a reverse indicator bar graph (higher rotation → fewer LEDs lit).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LEDs stay ON all the time | Potentiometer wired incorrectly | Check that the center pin of the potentiometer is wired to GP26, and the outer pins go to GND and 3.3V. |
| LEDs do not light up | Cathode grounding issue | Ensure the cathode (ground connection) for all three LEDs is firmly connected to GND. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [11 - Pico Potentiometer Serial](11-pico-potentiometer-serial.md)
- [17 - Pico LED Brightness Knob](17-pico-led-brightness-knob.md)
- [30 - Pico Light Level Meter LEDs](30-pico-light-level-meter-leds.md)
