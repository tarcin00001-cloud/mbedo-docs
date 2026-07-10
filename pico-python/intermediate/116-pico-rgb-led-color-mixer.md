# 116 - Pico RGB LED Color Mixer

Build a three-channel RGB LED colour mixer console that reads three independent potentiometers to control the red, green, and blue channels of a common-cathode RGB LED.

## Goal
Learn how to read three analog potentiometers on separate ADC channels, map their values to PWM duty cycles, and drive each colour channel of an RGB LED independently in MicroPython.

## What You Will Build
An RGB colour mixing station:
- **Potentiometer 1 (GP26)**: Controls the Red channel intensity.
- **Potentiometer 2 (GP27)**: Controls the Green channel intensity.
- **Potentiometer 3 (GP28)**: Controls the Blue channel intensity.
- **RGB LED (GP10, GP11, GP12)**: Displays the mixed colour in real-time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Potentiometer × 3 | `potentiometer` | Yes | Yes |
| Common-Cathode RGB LED | `led` | Yes (three separate LEDs) | Yes |
| 100 Ω Resistors × 3 | `resistor` | Optional in MbedO | Yes — one per LED channel |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer 1 (Red) | Wiper (centre) | GP26 | Red | Red channel ADC input |
| Potentiometer 2 (Green) | Wiper (centre) | GP27 | Green | Green channel ADC input |
| Potentiometer 3 (Blue) | Wiper (centre) | GP28 | Blue | Blue channel ADC input |
| All Potentiometers | Left pin | GND | Black | Ground reference |
| All Potentiometers | Right pin | 3.3V (3V3) | Red | High voltage reference |
| RGB LED — Red pin | Anode (via 100 Ω resistor) | GP10 | Red | Red PWM output |
| RGB LED — Green pin | Anode (via 100 Ω resistor) | GP11 | Green | Green PWM output |
| RGB LED — Blue pin | Anode (via 100 Ω resistor) | GP12 | Blue | Blue PWM output |
| RGB LED | Common Cathode (−) | GND | Black | Shared return path to ground |

> **Wiring tip:** All three potentiometer wipers connect to separate ADC pins (GP26, GP27, GP28). Connect each LED colour channel to a separate PWM GPIO through a 100 Ω current-limiting resistor. All grounds are shared.

## Code
```python
from machine import Pin, ADC, PWM
import utime

# Three analog inputs — one per colour channel
adc_r = ADC(26) # GP26 = Red channel
adc_g = ADC(27) # GP27 = Green channel
adc_b = ADC(28) # GP28 = Blue channel

# Three PWM outputs — one per colour channel
pwm_r = PWM(Pin(10))
pwm_g = PWM(Pin(11))
pwm_b = PWM(Pin(12))

pwm_r.freq(1000)
pwm_g.freq(1000)
pwm_b.freq(1000)

print("RGB Color Mixer active.")

while True:
    # Read raw ADC values (0 - 65535 on 16-bit scale)
    raw_r = adc_r.read_u16()
    raw_g = adc_g.read_u16()
    raw_b = adc_b.read_u16()
    
    # Write directly to PWM duty cycle (same 16-bit scale)
    pwm_r.duty_u16(raw_r)
    pwm_g.duty_u16(raw_g)
    pwm_b.duty_u16(raw_b)
    
    # Scale to 0-255 for display
    r8 = raw_r >> 8
    g8 = raw_g >> 8
    b8 = raw_b >> 8
    
    print("R:{:3d} G:{:3d} B:{:3d} | #{:02X}{:02X}{:02X}".format(
        r8, g8, b8, r8, g8, b8))
    
    utime.sleep_ms(100)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **three Potentiometers**, and **three LEDs** (representing R, G, B channels) onto the canvas.
2. Connect Potentiometers to **GP26**, **GP27**, **GP28**. Connect Red LED to **GP10**, Green to **GP11**, Blue to **GP12**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Adjust each potentiometer slider and observe the corresponding LED brightness change.

## Expected Output
```
RGB Color Mixer active.
R:128 G: 64 B:255 | #8040FF
R:255 G:255 B:  0 | #FFFF00
```

## Expected Canvas Behavior
- Each LED component on the canvas changes its glow brightness independently in response to its corresponding potentiometer slider.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `pwm_r.duty_u16(raw_r)` | Sets the PWM duty cycle of the red channel directly to the 16-bit ADC reading (both share the same scale). |
| `raw_r >> 8` | Right-shifts the 16-bit value by 8 bits to convert it to a 0–255 range for display. |
| `"#{:02X}{:02X}{:02X}".format(r8, g8, b8)` | Formats the three channel values as a CSS-style hex colour code. |

## Hardware & Safety Concept: PWM and Human Visual Perception
Human eyes cannot perceive individual flashes above approximately 60 Hz. PWM frequencies above this threshold create the perception of continuous, smoothly dimming light. The Pico's 1000 Hz PWM frequency is well above the human flicker-fusion threshold, ensuring a flicker-free colour mixing experience at all brightness levels.

## Try This! (Challenges)
1. **Preset Colours**: Add three buttons on GP13-GP15 that jump to preset colours (Red, Green, White) when pressed.
2. **OLED Display**: Add an OLED on GP4/GP5 that displays the current hex colour code.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| One colour channel does not respond | ADC pin conflict | Verify each potentiometer is wired to a separate ADC-capable pin (GP26, GP27, GP28). |
| LED too dim at maximum | Resistor value too high | Use 100 Ω instead of 330 Ω for brighter output. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [05 - Pico LED Fade PWM](05-pico-led-fade-pwm.md)
- [26 - Pico RGB LED Fixed Colors](26-pico-rgb-led-fixed-colors.md)
- [27 - Pico RGB LED Potentiometer](27-pico-rgb-led-potentiometer.md)
