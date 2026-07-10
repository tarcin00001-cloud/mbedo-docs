# 01 - Pico LED Blink

Make a single LED blink on and off repeatedly — the classic first program for any microcontroller.

## Goal
Learn how to configure a GPIO pin as a digital output and use `utime.sleep()` to create a timed on/off loop in MicroPython.

## What You Will Build
A blinking LED circuit:
- **LED (GP15)**: Turns ON and OFF every second.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LED (any colour) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes — required on hardware to limit current |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED | Anode (+, longer leg) | GP15 | Red | Signal pin — drives LED HIGH/LOW |
| LED | Cathode (−, shorter leg) | GND | Black | Return path to ground |
| 330 Ω Resistor | Either leg | In series between GP15 and LED Anode | — | Limits current to ~10 mA; omit in MbedO simulation |

> **Wiring tip:** On real hardware, always place a current-limiting resistor in series with the LED. Without it, excess current will burn out the LED and may damage the Pico's GPIO pin.

## Code
```python
from machine import Pin
import utime

led = Pin(15, Pin.OUT)

while True:
    led.value(1)      # Turn LED ON
    utime.sleep(1)    # Wait 1 second
    led.value(0)      # Turn LED OFF
    utime.sleep(1)    # Wait 1 second
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and an **LED** onto the canvas.
2. Connect the LED Anode to **GP15** and Cathode to **GND**.
3. Paste the code above into the code editor, select **MicroPython** mode, and click **Run**.

## Expected Output
The LED blinks ON for 1 second, then OFF for 1 second, repeating continuously.

## Expected Canvas Behavior
- The LED component on the canvas glows and dims alternately every second.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Pin(15, Pin.OUT)` | Configures GP15 as a digital output pin. |
| `led.value(1)` | Drives GP15 HIGH (3.3 V), turning the LED ON. |
| `utime.sleep(1)` | Pauses execution for 1 second before moving to the next line. |
| `led.value(0)` | Drives GP15 LOW (0 V), turning the LED OFF. |

## Hardware & Safety Concept: GPIO Current Limits
Each GPIO pin on the Raspberry Pi Pico can safely source or sink a maximum of **12 mA**. Always use a current-limiting resistor (typically 330 Ω for a 3.3 V supply) to keep LED current within this safe range. Omitting the resistor can permanently damage the pin.

## Try This! (Challenges)
1. **Faster Blink**: Change both `sleep(1)` calls to `sleep(0.2)` to blink five times per second.
2. **Asymmetric Pulse**: Use `sleep(0.1)` for ON and `sleep(0.9)` for OFF to create a brief flash effect.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays OFF | Anode/Cathode reversed | Flip the LED — the longer leg is the Anode (+). |
| LED glows dimly at all times | Floating GND connection | Ensure the Cathode is connected firmly to a GND pin. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**. No compilation step is needed — click Run and the code executes immediately on the virtual Pico.

## Related Projects
- [02 - Pico Double Blink](02-pico-double-blink.md)
- [04 - Pico PWM LED Fade](04-pico-pwm-led.md)
