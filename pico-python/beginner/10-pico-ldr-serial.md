# 10 - Pico LDR Light Sensor Serial

Read the ambient light level from an LDR (Light Dependent Resistor) and print raw ADC values to the serial monitor.

## Goal
Learn how to use `machine.ADC` to read an analog voltage on GP26 and print the resulting light level values to the MbedO serial terminal in MicroPython.

## What You Will Build
A light level monitor:
- **LDR (GP26)**: Reads ambient light intensity and prints values to the serial terminal every second.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LDR Photoresistor | `ldr` | Yes | Yes |
| 10 kΩ Pull-down Resistor | `resistor` | Optional in MbedO | Yes — required to form a voltage divider |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR | Pin 1 (either leg) | 3.3V (3V3) | Red | One LDR leg connected to the 3.3 V supply rail |
| LDR | Pin 2 (other leg) | GP26 | Yellow | ADC input reads the mid-point voltage of the divider |
| 10 kΩ Resistor | Pin 1 | GP26 | Yellow | Top of pull-down resistor — connects to the LDR/ADC junction |
| 10 kΩ Resistor | Pin 2 | GND | Black | Bottom of pull-down resistor — connects to ground |

> **Wiring tip:** The LDR and 10 kΩ resistor form a **voltage divider** between 3.3 V and GND. In bright light the LDR resistance drops (e.g. 1 kΩ), so the ADC pin voltage rises toward 3.3 V. In darkness the LDR resistance rises (e.g. 1 MΩ), pulling the ADC pin voltage toward 0 V.
>
> **ADC pins on the Pico:** Only GP26, GP27, GP28, and GP29 are ADC-capable. Do NOT use other GP pins for analog input — they will only read digital HIGH/LOW.

## Code
```python
from machine import ADC
import utime

ldr = ADC(26)     # GP26 = ADC channel 0

while True:
    raw = ldr.read_u16()          # 16-bit value: 0 (dark) to 65535 (bright)
    voltage = raw * 3.3 / 65535   # Convert to voltage in volts

    print("Light ADC:", raw, "| Voltage:", round(voltage, 2), "V")
    utime.sleep(1)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and an **LDR** onto the canvas.
2. Connect LDR Pin 1 to **3.3V** and Pin 2 to **GP26**. In MbedO, the pull-down resistor is built into the LDR component.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Adjust the LDR slider on the canvas to simulate different light levels and watch the values change.

## Expected Output
```
Light ADC: 52400 | Voltage: 2.64 V
Light ADC: 48000 | Voltage: 2.41 V
Light ADC: 10200 | Voltage: 0.51 V
```

## Expected Canvas Behavior
- Moving the LDR slider up (bright) increases the ADC value toward 65535.
- Moving the LDR slider down (dark) decreases the ADC value toward 0.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `ADC(26)` | Creates an ADC object mapped to GP26 (ADC channel 0). |
| `ldr.read_u16()` | Reads the current ADC voltage as a 16-bit unsigned integer (0 to 65535). |
| `raw * 3.3 / 65535` | Converts the raw 16-bit ADC value to the actual voltage at GP26 in volts. |
| `print(...)` | Outputs the values to the MbedO serial terminal on each loop iteration. |

## Hardware & Safety Concept: Voltage Dividers
A voltage divider uses two resistors in series between supply and ground. The output voltage at the midpoint is proportional to the ratio of the resistors. When one resistor is a variable element (like an LDR, NTC thermistor, or potentiometer), the output voltage changes in response to the physical measurement, making it readable by an ADC pin.

## Try This! (Challenges)
1. **Day/Night LED**: Add an LED on GP15 that turns ON automatically when the LDR reading drops below 20000 (dark room).
2. **Percentage Display**: Map the ADC reading to a 0–100% brightness percentage and print that instead of the raw value.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| ADC always reads 0 | LDR not connected to 3.3V | Check that one LDR leg is connected to the 3.3V pin, not GND. |
| ADC always reads 65535 | Pull-down resistor missing | Without the 10 kΩ pull-down to GND, the pin floats HIGH. Add the resistor in series to GND. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**. The serial `print()` output appears in the MbedO terminal panel.

## Related Projects
- [11 - Pico Potentiometer Serial](11-pico-potentiometer-serial.md)
- [12 - Pico NTC Thermistor Serial](12-pico-ntc-serial.md)
