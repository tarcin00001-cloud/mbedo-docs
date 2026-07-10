# 04 - Pico PWM LED Fade

Smoothly fade an LED from completely off to fully bright and back using Pulse Width Modulation (PWM).

## Goal
Learn how to use `machine.PWM` to generate a variable duty-cycle signal on a GPIO pin, creating a smooth LED brightness fade effect in MicroPython.

## What You Will Build
A breathing LED circuit:
- **LED (GP15)**: Fades from 0% to 100% brightness and back in a continuous loop.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LED (any colour) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes — for LED current limiting |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED | Anode (+, longer leg) | GP15 | Red | PWM output pin — carries the variable duty-cycle signal |
| LED | Cathode (−, shorter leg) | GND | Black | LED return path to ground |
| 330 Ω Resistor | Either leg | In series between GP15 and LED Anode | — | Limits peak current to ~10 mA at 100% duty cycle |

> **Wiring tip:** Any GPIO pin on the Pico can generate PWM. GP15 is used here, but any GP pin from GP0 to GP28 will work equally well.

## Code
```python
from machine import Pin, PWM
import utime

pwm_led = PWM(Pin(15))
pwm_led.freq(1000)          # Set PWM frequency to 1 kHz

while True:
    # Fade IN (0% → 100%)
    for duty in range(0, 65536, 512):
        pwm_led.duty_u16(duty)
        utime.sleep_ms(10)

    # Fade OUT (100% → 0%)
    for duty in range(65535, -1, -512):
        pwm_led.duty_u16(duty)
        utime.sleep_ms(10)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and an **LED** onto the canvas.
2. Connect LED Anode to **GP15** and Cathode to **GND**.
3. Paste the code, select **MicroPython** mode, and click **Run**.

## Expected Output
The LED gradually brightens from off to full brightness, then dims back to off, cycling continuously in a breathing pattern.

## Expected Canvas Behavior
- The LED component on the canvas smoothly brightens and dims in a continuous breath-like pattern.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `PWM(Pin(15))` | Attaches the PWM generator to GP15. |
| `pwm_led.freq(1000)` | Sets the PWM carrier frequency to 1000 Hz (1 kHz), fast enough that the eye sees smooth brightness rather than flicker. |
| `pwm_led.duty_u16(duty)` | Sets the duty cycle using a 16-bit value: `0` = 0% (fully off), `65535` = 100% (fully on). |
| `range(0, 65536, 512)` | Steps the duty cycle up in 128 increments of 512 from 0 to 65535. |
| `utime.sleep_ms(10)` | Waits 10 ms between steps — full fade takes about 1.3 seconds. |

## Hardware & Safety Concept: Pulse Width Modulation
PWM works by switching the output pin ON and OFF very rapidly. The ratio of ON time to total cycle time is the **duty cycle**. At 50% duty, the LED appears at half brightness because it is ON for exactly half of each millisecond cycle — too fast for the human eye to perceive as flicker.

## Try This! (Challenges)
1. **Speed Control**: Change `sleep_ms(10)` to `sleep_ms(5)` to make the fade twice as fast.
2. **Sine Wave Fade**: Replace the linear `range()` loop with a sine wave lookup for a smoother, more organic breathing pattern.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays at a constant brightness | PWM duty cycle not changing | Verify the `for` loop is correctly iterating the `duty` variable and calling `duty_u16()` inside the loop. |
| LED flickers visibly | PWM frequency too low | Increase `freq()` to at least `500` Hz — ideally `1000` Hz or higher. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**. PWM is supported natively in the virtual Pico environment.

## Related Projects
- [01 - Pico LED Blink](01-pico-blink.md)
- [05 - Pico RGB LED](05-pico-rgb-led.md)
