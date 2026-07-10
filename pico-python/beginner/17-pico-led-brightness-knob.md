# 17 - Pico LED Brightness Knob

Control the brightness of an LED using a rotary potentiometer knob.

## Goal
Learn how to map an analog ADC input (0–65535) directly to a PWM duty cycle (0–65535) to control LED brightness linearly in MicroPython.

## What You Will Build
An analog dimmer circuit:
- **Potentiometer (GP26)**: Measures rotation angle as an analog voltage.
- **LED (GP15)**: Adjusts brightness smoothly from 0% (off) to 100% (full on) based on the knob's position.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Potentiometer (any value) | `potentiometer` | Yes | Yes |
| LED (any colour) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes — for LED current limiting |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Left Pin (Pin 1) | GND | Black | Ground reference |
| Potentiometer | Wiper (Pin 2 - centre) | GP26 | Yellow | Analog input (reads 0V to 3.3V) |
| Potentiometer | Right Pin (Pin 3) | 3.3V (3V3) | Red | High voltage reference |
| LED | Anode (+, longer leg) | GP15 | Orange | PWM output signal |
| LED | Cathode (−, shorter leg) | GND | Black | Shared return path to ground |
| 330 Ω Resistor | Either leg | In series between GP15 and LED Anode | — | Limits LED current to ~10 mA |

> **Wiring tip:** Connect the outer terminals of the potentiometer to 3.3V and GND, and the middle pin (wiper) to GP26. The LED anode connects to GP15 through the current-limiting resistor.

## Code
```python
from machine import Pin, ADC, PWM
import utime

pot = ADC(26)        # GP26 = ADC channel 0
led = PWM(Pin(15))   # GP15 = PWM output
led.freq(1000)       # Set PWM frequency to 1 kHz

while True:
    # Read analog value (0 - 65535)
    val = pot.read_u16()
    
    # Write directly to PWM duty cycle (0 - 65535)
    led.duty_u16(val)
    
    utime.sleep_ms(20) # Smooth updates
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Potentiometer**, and **LED** onto the canvas.
2. Connect Potentiometer Pin 1 to **GND**, Pin 2 (wiper) to **GP26**, and Pin 3 to **3.3V**.
3. Connect LED Anode to **GP15** and Cathode to **GND**.
4. Paste the code, select **MicroPython** mode, and click **Run**.
5. Adjust the potentiometer slider on the canvas to dim or brighten the LED.

## Expected Output
The LED brightness increases continuously as you slide the potentiometer up, and dims down to off when you slide it down.

## Expected Canvas Behavior
- The LED component on the canvas smoothly brightens and dims in real-time correlation with the potentiometer slider movements.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `ADC(26)` | Configures GP26 as an analog input to read the wiper voltage. |
| `PWM(Pin(15))` | Attaches the PWM generator to GP15. |
| `pot.read_u16()` | Returns a 16-bit integer (0–65535) representing 0V to 3.3V. |
| `led.duty_u16(val)` | Sends the raw ADC value directly to the PWM duty cycle register, matching the input scale. |

## Hardware & Safety Concept: Direct ADC-to-PWM Mapping
Because both the Raspberry Pi Pico's ADC reads and PWM duty configurations use a **16-bit unsigned integer range** (0 to 65535) in MicroPython, we can feed the output of `pot.read_u16()` directly into `led.duty_u16()`. This avoids complex mathematical scaling equations and minimizes processing overhead.

## Try This! (Challenges)
1. **Logarithmic Dimming**: Human eyes perceive brightness logarithmically rather than linearly. Apply a scaling formula to make the dimming curve feel more natural.
2. **Reverse Dimmer**: Adjust the code so that rotating the potentiometer clockwise dims the LED instead of brightening it.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED flashes or strobes | PWM frequency set too low | Ensure `led.freq(1000)` is defined in `setup` to keep the frequency above human flicker thresholds. |
| LED stays ON at full brightness | Wiper connected to outer pin | Ensure the center pin of the potentiometer is wired to GP26. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [04 - Pico PWM LED Fade](04-pico-pwm-led.md)
- [11 - Pico Potentiometer Serial](11-pico-potentiometer-serial.md)
