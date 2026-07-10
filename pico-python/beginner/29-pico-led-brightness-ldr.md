# 29 - Pico LED Brightness LDR

Adjust an LED's brightness automatically based on ambient light levels (dimmer room → brighter LED).

## Goal
Learn how to invert analog light sensor (LDR) readings and map them to a PWM duty cycle to control LED brightness dynamically in MicroPython.

## What You Will Build
An smart ambient backlight controller:
- **LDR Sensor (GP26)**: Measures ambient room lighting.
- **LED (GP15)**: Automatically brightens in the dark to act as a night light, and dims to 0% (off) in bright daylight.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LDR Photoresistor | `ldr` | Yes | Yes |
| LED (any colour) | `led` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | Optional in MbedO | Yes (voltage divider) |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR | Pin 1 | 3.3V (3V3) | Red | Power line |
| LDR | Pin 2 | GP26 | Yellow | Analog input signal |
| 10 kΩ Resistor | Either leg | GP26 to GND | Black | Pull-down resistor |
| LED | Anode (+, longer leg) | GP15 | Orange | PWM control signal |
| LED | Cathode (−, shorter leg) | GND | Black | Shared return path to ground |
| 330 Ω Resistor | Either leg | In series with LED Anode | — | Limits LED current |

> **Wiring tip:** The LDR and 10 kΩ resistor form a voltage divider connected to GP26. The LED anode connects to GP15 through the current-limiting resistor.

## Code
```python
from machine import Pin, ADC, PWM
import utime

ldr = ADC(26)      # GP26 = ADC channel 0
led = PWM(Pin(15)) # GP15 = PWM output
led.freq(1000)     # Set PWM frequency to 1 kHz

while True:
    raw = ldr.read_u16()
    
    # Invert logic: bright light (high raw) -> low LED brightness (low duty)
    #               dark room (low raw) -> high LED brightness (high duty)
    duty = 65535 - raw
    
    # Ensure duty remains within valid 16-bit range
    if duty < 0:
        duty = 0
    elif duty > 65535:
        duty = 65535
        
    led.duty_u16(duty)
    
    print("Ambient Light:", raw, "| LED Duty:", duty)
    utime.sleep_ms(100) # Fast updates
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **LDR**, and **LED** onto the canvas.
2. Connect LDR Pin 1 to **3.3V** and Pin 2 to **GP26**.
3. Connect LED Anode to **GP15** and Cathode to **GND**.
4. Paste the code, select **MicroPython** mode, and click **Run**.
5. Adjust the LDR slider on the canvas and observe the LED brightness changing.

## Expected Output
```
Ambient Light: 55000 | LED Duty: 10535
Ambient Light: 12000 | LED Duty: 53535
```

## Expected Canvas Behavior
- The LED component on the canvas glows brighter as the LDR slider is moved down (darker), and dims to off as the slider is moved up (brighter).

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `65535 - raw` | Inverts the LDR reading. When the input voltage is high (bright light), the calculated duty cycle becomes low (dim LED). |
| `led.duty_u16(duty)` | Applies the calculated duty cycle to GP15 to set the LED brightness. |

## Hardware & Safety Concept: Energy-Saving Backlights
Smart dimming backlights are widely used in smartphones, laptop screens, and automotive dashboard consoles. By automatically lowering screen brightness in dark environments and raising it in bright daylight, systems improve readability while saving battery energy.

## Try This! (Challenges)
1. **Linear Range Scaling**: Map specific minimum and maximum LDR light limits to the full 0–65535 PWM range, ignoring values outside those limits.
2. **Transition Smoothing**: Implement a software moving-average filter to smooth out sudden light changes (like hand shadows passing over the sensor).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays at full brightness | LDR pin disconnected | Ensure the LDR signal wire is firmly connected to GP26. |
| LED glows when it should be OFF | Calibration range offset | Adjust the inversion formula offset (e.g. `55000 - raw`) if the ambient light does not reach 65535. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [04 - Pico PWM LED Fade](04-pico-pwm-led.md)
- [10 - Pico LDR Light Sensor Serial](10-pico-ldr-serial.md)
- [22 - Pico Night Light LDR](22-pico-night-light-ldr.md)
