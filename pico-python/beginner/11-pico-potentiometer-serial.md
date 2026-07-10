# 11 - Pico Potentiometer Serial

Read the wiper position of a potentiometer using the ADC and print the value and equivalent voltage to the serial terminal.

## Goal
Learn how to read a continuously variable analog signal from a potentiometer using `machine.ADC`, convert the raw reading to a percentage and voltage, and print it to the serial monitor in MicroPython.

## What You Will Build
A rotary position reader:
- **Potentiometer (GP26)**: Reads wiper position and prints ADC value, voltage, and percentage to the terminal every 500 ms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Potentiometer (any value 1 kΩ – 100 kΩ) | `potentiometer` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Left terminal (Pin 1) | GND | Black | Low reference voltage — wiper reads 0 V at this end |
| Potentiometer | Wiper (Pin 2 — centre) | GP26 | Yellow | ADC input — reads voltage proportional to wiper position |
| Potentiometer | Right terminal (Pin 3) | 3.3V (3V3) | Red | High reference voltage — wiper reads 3.3 V at this end |

> **Wiring tip:** The potentiometer acts as an adjustable voltage divider. Rotating the shaft moves the wiper between GND (0 V) and 3.3 V. The ADC on GP26 reads the wiper voltage directly — no additional pull-up or pull-down resistor is needed.
>
> **Orientation:** Swapping Pin 1 and Pin 3 reverses the direction — maximum rotation reads near 0 instead of 65535. Either orientation is electrically safe.

## Code
```python
from machine import ADC
import utime

pot = ADC(26)    # GP26 = ADC channel 0

while True:
    raw     = pot.read_u16()              # 0 to 65535
    voltage = raw * 3.3 / 65535          # Convert to volts
    percent = raw * 100 // 65535         # Convert to 0–100%

    print("ADC:", raw,
          "| Voltage:", round(voltage, 2), "V",
          "| Position:", percent, "%")
    utime.sleep_ms(500)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and a **Potentiometer** onto the canvas.
2. Connect Pin 1 (Left terminal) to **GND**, Pin 2 (Wiper) to **GP26**, Pin 3 (Right terminal) to **3.3V**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Drag the potentiometer slider on the canvas and watch the values update in the terminal.

## Expected Output
```
ADC: 0     | Voltage: 0.0 V  | Position: 0 %
ADC: 32767 | Voltage: 1.65 V | Position: 50 %
ADC: 65535 | Voltage: 3.3 V  | Position: 100 %
```

## Expected Canvas Behavior
- Moving the canvas potentiometer slider from left to right increases the ADC reading linearly from 0 to 65535.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `ADC(26)` | Creates an ADC object mapped to GP26 (channel 0). |
| `pot.read_u16()` | Returns a 16-bit reading (0–65535) representing the wiper voltage from 0 V to 3.3 V. |
| `raw * 3.3 / 65535` | Converts the 16-bit reading to the actual voltage at the wiper. |
| `raw * 100 // 65535` | Converts the reading to a 0–100% percentage using integer division. |

## Hardware & Safety Concept: ADC Reference Voltage
The Pico's ADC uses the 3.3 V supply as its reference (VREF). Any voltage applied to an ADC pin must stay within 0 V to 3.3 V — **never apply a voltage higher than 3.3 V to an ADC pin**. Exceeding this limit will permanently damage the ADC input, even if the Pico is powered from 5V via USB.

## Try This! (Challenges)
1. **LED Brightness Control**: Wire an LED to GP15 with a 330 Ω resistor and use the potentiometer reading to set the LED's PWM brightness level.
2. **Threshold Buzzer**: Sound a buzzer on GP14 when the potentiometer passes 75% (ADC > 49151).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| ADC reads 0 at all positions | Wiper not connected to GP26 | Check that Pin 2 (centre wiper) is connected to GP26, not one of the outer terminals. |
| ADC reads maximum at all positions | Wiper connected to 3.3V terminal | Ensure Pin 2 is the wiper, not the power rail. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**. The serial output appears in the MbedO terminal panel.

## Related Projects
- [10 - Pico LDR Light Sensor Serial](10-pico-ldr-serial.md)
- [12 - Pico NTC Thermistor Serial](12-pico-ntc-serial.md)
