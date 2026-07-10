# 05 - Pico RGB LED

Mix red, green, and blue channels on an RGB LED using independent PWM signals to produce any colour.

## Goal
Learn how to drive a common-cathode RGB LED using three separate PWM channels, and understand additive colour mixing to produce custom colours in MicroPython.

## What You Will Build
A colour-cycling RGB LED:
- **Red channel (GP13)**: Controls the red LED element.
- **Green channel (GP14)**: Controls the green LED element.
- **Blue channel (GP15)**: Controls the blue LED element.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| RGB LED (common cathode) | `led_rgb` | Yes | Yes |
| 330 Ω Resistors × 3 | `resistor` | Optional in MbedO | Yes — one per colour channel |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| RGB LED | Red Anode (R) | GP13 | Red | PWM channel for red — 330 Ω resistor in series |
| RGB LED | Green Anode (G) | GP14 | Green | PWM channel for green — 330 Ω resistor in series |
| RGB LED | Blue Anode (B) | GP15 | Blue | PWM channel for blue — 330 Ω resistor in series |
| RGB LED | Common Cathode (longest leg or marked K) | GND | Black | Shared ground for all three colour channels |

> **Wiring tip:** Common-cathode RGB LEDs have **four legs**. The longest leg is the shared cathode (GND). Place a separate 330 Ω resistor in series with each of the three colour anodes — not one shared resistor — so each channel can be independently dimmed.

> **Common-anode variant:** If your RGB LED has a common anode, connect the common pin to 3.3 V and invert the duty logic: `duty_u16(0)` = full brightness, `duty_u16(65535)` = off.

## Code
```python
from machine import Pin, PWM
import utime

red   = PWM(Pin(13))
green = PWM(Pin(14))
blue  = PWM(Pin(15))

red.freq(1000)
green.freq(1000)
blue.freq(1000)

def set_colour(r, g, b):
    """Set RGB values 0-255."""
    red.duty_u16(int(r / 255 * 65535))
    green.duty_u16(int(g / 255 * 65535))
    blue.duty_u16(int(b / 255 * 65535))

while True:
    set_colour(255, 0, 0)    # Red
    utime.sleep(1)
    set_colour(0, 255, 0)    # Green
    utime.sleep(1)
    set_colour(0, 0, 255)    # Blue
    utime.sleep(1)
    set_colour(255, 255, 0)  # Yellow
    utime.sleep(1)
    set_colour(0, 255, 255)  # Cyan
    utime.sleep(1)
    set_colour(255, 0, 255)  # Magenta
    utime.sleep(1)
    set_colour(255, 255, 255) # White
    utime.sleep(1)
    set_colour(0, 0, 0)      # Off
    utime.sleep(1)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and an **RGB LED** onto the canvas.
2. Connect R Anode to **GP13**, G Anode to **GP14**, B Anode to **GP15**, and Common Cathode to **GND**.
3. Paste the code, select **MicroPython** mode, and click **Run**.

## Expected Output
The LED cycles through Red → Green → Blue → Yellow → Cyan → Magenta → White → Off, holding each colour for 1 second.

## Expected Canvas Behavior
- The RGB LED component on the canvas changes colour to match each `set_colour()` call.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `PWM(Pin(13))` | Attaches a PWM generator to GP13 for the red channel. |
| `set_colour(r, g, b)` | Helper function that converts 0–255 colour values to 16-bit PWM duty cycles. |
| `int(r / 255 * 65535)` | Maps the 0–255 range to the 0–65535 16-bit duty cycle range. |
| `set_colour(255, 255, 0)` | Yellow is produced by mixing full red and full green — no blue. |

## Hardware & Safety Concept: Additive Colour Mixing
RGB LEDs use **additive mixing**: combining red + green produces yellow, red + blue produces magenta, green + blue produces cyan, and all three at full brightness produce white. This is the same principle used by television and computer display screens.

## Try This! (Challenges)
1. **Smooth Colour Fade**: Write a loop that continuously shifts the hue by incrementing R, G, and B in a rotating pattern.
2. **Potentiometer Colour Control**: Wire a potentiometer to GP26 and map its ADC value to the red channel brightness.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| One colour channel never lights | Resistor or wire missing on that channel | Check that all three colour anodes have their own resistor and connection to the correct GP pin. |
| Colours look wrong (e.g. yellow looks orange) | Common-anode LED with common-cathode code | Invert the logic: connect common to 3.3 V and swap `0` and `65535` in `set_colour()`. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**. All three PWM channels are supported simultaneously.

## Related Projects
- [04 - Pico PWM LED Fade](04-pico-pwm-led.md)
- [09 - Pico Traffic Light](09-pico-traffic-light.md)
