# 22 - Pico Night Light LDR

Build an automatic night light that turns an LED ON when the room gets dark, and OFF when it is bright.

## Goal
Learn how to read an analog light sensor (LDR) and implement threshold logic to control a digital output pin in MicroPython.

## What You Will Build
An automatic night light:
- **LDR Sensor (GP26)**: Measures ambient light levels.
- **LED (GP15)**: Automatically turns ON when the light level drops below a set threshold, and OFF when the light level is high.

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
| LDR | Pin 2 | GP26 | Yellow | Analog signal output |
| 10 kΩ Resistor | Either leg | GP26 to GND | Black | Pull-down resistor |
| LED | Anode (+, longer leg) | GP15 | Orange | Control signal pin |
| LED | Cathode (−, shorter leg) | GND | Black | Shared return path to ground |
| 330 Ω Resistor | Either leg | In series with LED Anode | — | Limits LED current |

> **Wiring tip:** The LDR and 10 kΩ resistor form a voltage divider connected to GP26. In bright light, GP26 reads a high voltage (large ADC value). In dark conditions, GP26 reads a low voltage (small ADC value).

## Code
```python
from machine import Pin, ADC
import utime

ldr = ADC(26) # GP26 = ADC channel 0
led = Pin(15, Pin.OUT)

# Threshold: values below this indicate darkness (raw ADC range 0 - 65535)
DARK_LIMIT = 20000

while True:
    val = ldr.read_u16()
    print("Light Level:", val)

    if val < DARK_LIMIT:
        led.value(1) # Turn LED ON (Dark)
    else:
        led.value(0) # Turn LED OFF (Bright)

    utime.sleep_ms(200) # Update rate
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **LDR**, and **LED** onto the canvas.
2. Connect LDR Pin 1 to **3.3V** and Pin 2 to **GP26**.
3. Connect LED Anode to **GP15** and Cathode to **GND**.
4. Paste the code, select **MicroPython** mode, and click **Run**.
5. Move the LDR slider down (dark) to see the LED turn ON, and slide it up (bright) to see it turn OFF.

## Expected Output
```
Light Level: 45000
Light Level: 12000
```

## Expected Canvas Behavior
- The LED component on the canvas lights up when the LDR slider is moved down past the threshold.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `val < DARK_LIMIT` | Checks if the measured light level is below the threshold, indicating dark conditions. |
| `led.value(1)` | Turns the night light ON when darkness is detected. |

## Hardware & Safety Concept: Hysteresis
Simple threshold logic can cause the light to flicker if the sensor reading hovers right around the limit. To prevent this, commercial systems use **hysteresis**: they turn the light ON at a lower threshold (e.g. 18000) and OFF at a higher threshold (e.g. 22000).

## Try This! (Challenges)
1. **Implement Hysteresis**: Modify the code to include separate turn-ON and turn-OFF thresholds to prevent flickering.
2. **Reverse Night Light**: Configure the circuit to turn the LED ON in bright light and OFF in the dark (solar tracker mode).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays ON constantly | Threshold too high | Print LDR values to the terminal to find the ambient light level, then adjust the `DARK_LIMIT` in your code. |
| LED does not turn ON in the dark | LDR wired incorrectly | Verify that the LDR and fixed resistor are configured as a voltage divider at GP26. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [10 - Pico LDR Light Sensor Serial](10-pico-ldr-serial.md)
- [17 - Pico LED Brightness Knob](17-pico-led-brightness-knob.md)
