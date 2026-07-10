# 37 - Pico RGB LED Fade

Fade an RGB LED automatically through the full color spectrum (Red → Yellow → Green → Cyan → Blue → Magenta → Red) using a smooth breathing cycle.

## Goal
Learn how to use mathematical equations (sine/cosine curves) to generate smooth, overlapping duty-cycle transitions across three separate PWM output channels in MicroPython.

## What You Will Build
A color-breathing RGB LED:
- **RGB LED (GP13, GP14, GP15)**: Smoothly transitions through the color wheel using sine-wave value scaling, avoiding harsh color steps.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| RGB LED (common cathode) | `led_rgb` | Yes | Yes |
| 330 Ω Resistors × 3 | `resistor` | Optional in MbedO | Yes — one per color channel |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| RGB LED | Red Anode (R) | GP13 | Red | Red channel PWM signal |
| RGB LED | Green Anode (G) | GP14 | Green | Green channel PWM signal |
| RGB LED | Blue Anode (B) | GP15 | Blue | Blue channel PWM signal |
| RGB LED | Common Cathode (longest leg) | GND | Black | Shared return path to ground |
| 330 Ω Resistors | Either leg | In series with R/G/B legs | — | Protects each color channel |

> **Wiring tip:** Connect the three RGB LED color pins to GP13, GP14, and GP15 respectively through 330 Ω resistors. The longest leg of the RGB LED connects to GND.

## Code
```python
from machine import Pin, PWM
import utime
import math

r_pwm = PWM(Pin(13))
g_pwm = PWM(Pin(14))
b_pwm = PWM(Pin(15))

r_pwm.freq(1000)
g_pwm.freq(1000)
b_pwm.freq(1000)

def set_color_duty(r_duty, g_duty, b_duty):
    r_pwm.duty_u16(r_duty)
    g_pwm.duty_u16(g_duty)
    b_pwm.duty_u16(b_duty)

print("Starting smooth RGB color cycle.")

# Angle counter for sine waves
angle = 0.0

while True:
    # Calculate phase-shifted sine waves for Red, Green, Blue
    # Mapping sine range (-1.0 to 1.0) to PWM duty range (0 to 65535)
    r_val = int((math.sin(angle) + 1.0) * 32767)
    g_val = int((math.sin(angle + 2.094) + 1.0) * 32767) # Phase shift 120 degrees (2.094 radians)
    b_val = int((math.sin(angle + 4.188) + 1.0) * 32767) # Phase shift 240 degrees (4.188 radians)
    
    set_color_duty(r_val, g_val, b_val)
    
    # Increment angle to step through the color wheel
    angle += 0.02
    if angle > 6.283: # Reset at 360 degrees (2 * PI radians)
        angle = 0.0
        
    utime.sleep_ms(20) # Smooth animation speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **RGB LED** onto the canvas.
2. Connect R Anode to **GP13**, G Anode to **GP14**, B Anode to **GP15**, and Common Cathode to **GND**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Observe the smooth color transitions.

## Expected Output
```
Starting smooth RGB color cycle.
```

## Expected Canvas Behavior
- The RGB LED component on the canvas transitions smoothly through the color wheel without any abrupt jumps or steps.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `math.sin(angle) + 1.0` | Shifts the sine wave output (normally -1.0 to 1.0) to a positive range (0.0 to 2.0). |
| `angle + 2.094` | Phase-shifts the Green channel's sine wave by 120 degrees (2.094 radians) so it peaks after Red. |
| `angle += 0.02` | Increments the step angle on each loop to animate the color transition. |

## Hardware & Safety Concept: Phase-Shifted Waveforms
Using phase-shifted sine waves (120 degrees apart) is the standard method for generating smooth color wheel rotations in RGB lighting. It ensures that the overall brightness of the LED remains relatively constant as the colors blend into each other, avoiding jarring changes in total light output.

## Try This! (Challenges)
1. **Speed Adjuster**: Connect a potentiometer on GP26 to adjust the speed of the color wheel rotation.
2. **Breathing Effect**: Add a second sine wave to scale all three color intensities together, creating a pulsing/breathing effect.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED flashes white instead of cycling colors | Phase shift values incorrect | Verify that the Green and Blue channel angles are phase-shifted correctly by `2.094` and `4.188` radians. |
| One color is missing from the cycle | Output pin wired incorrectly | Verify that GP13 (Red), GP14 (Green), and GP15 (Blue) are connected to the correct anodes. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**. The `math` library is built into the MicroPython environment.

## Related Projects
- [05 - Pico RGB LED](05-pico-rgb-led.md)
- [26 - Pico Potentiometer RGB Color](26-pico-potentiometer-rgb-color.md)
