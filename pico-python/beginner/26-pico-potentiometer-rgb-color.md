# 26 - Pico Potentiometer RGB Color

Adjust the color hues of an RGB LED smoothly using a rotary potentiometer knob.

## Goal
Learn how to read an analog potentiometer value (0–65535) and map it to distinct color sectors across Red, Green, and Blue PWM output channels in MicroPython.

## What You Will Build
An analog color controller:
- **Potentiometer (GP26)**: Reads knob adjustments.
- **RGB LED (GP13, GP14, GP15)**: Cycles through color hues (Red → Yellow → Green → Cyan → Blue → Violet → Red) based on the knob's position.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| RGB LED (common cathode) | `led_rgb` | Yes | Yes |
| 330 Ω Resistors × 3 | `resistor` | Optional in MbedO | Yes — one per color channel |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Left Pin (Pin 1) | GND | Black | Ground reference |
| Potentiometer | Wiper (Pin 2 - centre) | GP26 | Yellow | Analog input pin |
| Potentiometer | Right Pin (Pin 3) | 3.3V (3V3) | Red | High voltage reference |
| RGB LED | Red Anode (R) | GP13 | Red | Red channel PWM signal |
| RGB LED | Green Anode (G) | GP14 | Green | Green channel PWM signal |
| RGB LED | Blue Anode (B) | GP15 | Blue | Blue channel PWM signal |
| RGB LED | Common Cathode (longest leg) | GND | Black | Shared return path to ground |
| 330 Ω Resistors | Either leg | In series with R/G/B legs | — | Protects each color channel |

> **Wiring tip:** The potentiometer middle wiper pin connects to GP26. The three RGB LED color pins connect to GP13, GP14, and GP15 respectively through 330 Ω resistors. The longest leg of the RGB LED connects to GND.

## Code
```python
from machine import Pin, ADC, PWM
import utime

pot = ADC(26) # GP26 = ADC channel 0

r_pwm = PWM(Pin(13))
g_pwm = PWM(Pin(14))
b_pwm = PWM(Pin(15))

r_pwm.freq(1000)
g_pwm.freq(1000)
b_pwm.freq(1000)

def set_color(r, g, b):
    # Map 0-255 inputs to 16-bit PWM duty cycles (0 - 65535)
    r_pwm.duty_u16(int(r * 65535 / 255))
    g_pwm.duty_u16(int(g * 65535 / 255))
    b_pwm.duty_u16(int(b * 65535 / 255))

while True:
    raw = pot.read_u16()
    
    # Scale raw ADC (0-65535) to sector range (0 - 1535)
    hue = int(raw * 1535 / 65535)
    
    # Simple color wheel math (6 sectors of 256 steps each)
    if hue < 256: # Sector 1: Red to Yellow
        set_color(255, hue, 0)
    elif hue < 512: # Sector 2: Yellow to Green
        set_color(511 - hue, 255, 0)
    elif hue < 768: # Sector 3: Green to Cyan
        set_color(0, 255, hue - 512)
    elif hue < 1024: # Sector 4: Cyan to Blue
        set_color(0, 1023 - hue, 255)
    elif hue < 1280: # Sector 5: Blue to Violet
        set_color(hue - 1024, 0, 255)
    else: # Sector 6: Violet to Red
        set_color(255, 0, 1535 - hue)
        
    utime.sleep_ms(30)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Potentiometer**, and **RGB LED** onto the canvas.
2. Connect Potentiometer wiper to **GP26**. Connect RGB channels to **GP13**, **GP14**, and **GP15**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the potentiometer slider on the canvas and observe the RGB LED color changing.

## Expected Output
The RGB LED color cycles through the full spectrum (Red → Yellow → Green → Cyan → Blue → Violet) as you slide the potentiometer from bottom to top.

## Expected Canvas Behavior
- The RGB LED component on the canvas updates its color in real-time response to the position of the potentiometer slider.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `int(raw * 1535 / 65535)` | Translates the 16-bit ADC input into a 1536-step color wheel value (6 sectors × 256 levels of transition). |
| `set_color(255, hue, 0)` | Fades the Green channel up while keeping Red at maximum to shift the color from Red to Yellow. |

## Hardware & Safety Concept: Multi-Channel PWM Control
Microcontrollers use separate timers to generate independent PWM signals on different GPIO pins. This allows the system to dim three color channels (Red, Green, Blue) at the same time, producing different colors through additive mixing.

## Try This! (Challenges)
1. **Brightness Potentiometer**: Add a second potentiometer on GP27 to adjust the overall brightness of the selected color.
2. **Auto Cycle Mode**: Add a push button on GP12. Pressing it toggles between manual potentiometer color selection and automatic cycling.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Color changes are jerky or jumpy | Potentiometer wiper noise | Add a small capacitor (e.g. 100 nF) between the wiper pin and GND, or implement a software smoothing filter. |
| Some colors are missing from the cycle | One of the RGB channels is wired incorrectly | Verify the wiring for all three color anodes (Red on GP13, Green on GP14, Blue on GP15). |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [05 - Pico RGB LED](05-pico-rgb-led.md)
- [11 - Pico Potentiometer Serial](11-pico-potentiometer-serial.md)
- [17 - Pico LED Brightness Knob](17-pico-led-brightness-knob.md)
