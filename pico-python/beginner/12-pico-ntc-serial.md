# 12 - Pico NTC Thermistor Serial

Read temperature from an NTC thermistor using the ADC and print the converted temperature in Celsius to the serial terminal.

## Goal
Learn how to read an NTC thermistor voltage divider with `machine.ADC` and apply the Steinhart-Hart equation to convert the raw ADC value to a temperature in degrees Celsius in MicroPython.

## What You Will Build
A simple thermometer:
- **NTC Thermistor (GP26)**: Measures ambient temperature and prints it to the terminal every second.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| NTC Thermistor (10 kΩ at 25 °C) | `ntc` | Yes | Yes |
| 10 kΩ Fixed Resistor | `resistor` | Optional in MbedO | Yes — required for voltage divider |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| 10 kΩ Fixed Resistor | Pin 1 | 3.3V (3V3) | Red | Top of voltage divider — supply voltage |
| 10 kΩ Fixed Resistor | Pin 2 | GP26 | Yellow | Mid-point of divider — connects to ADC input and NTC |
| NTC Thermistor | Pin 1 | GP26 | Yellow | NTC connects ADC mid-point to GND |
| NTC Thermistor | Pin 2 | GND | Black | Bottom of voltage divider |

> **Wiring tip:** The 10 kΩ fixed resistor (R_fixed) and NTC thermistor form a voltage divider between 3.3 V and GND. As temperature rises, the NTC resistance drops, pulling the mid-point voltage DOWN toward GND. As temperature falls, NTC resistance rises, pulling the mid-point voltage UP toward 3.3 V.
>
> **Reading direction:** In this configuration (fixed resistor on top, NTC on bottom), cold temperature → high ADC value, hot temperature → low ADC value. The Steinhart-Hart formula in the code handles this conversion correctly.

## Code
```python
from machine import ADC
import utime
import math

ntc = ADC(26)       # GP26 = ADC channel 0

# Thermistor constants
R_FIXED  = 10000    # 10 kΩ fixed series resistor
R_NOM    = 10000    # NTC nominal resistance at 25 °C
T_NOM    = 25.0     # Nominal temperature in Celsius
B_COEFF  = 3950     # Beta coefficient for the thermistor (check datasheet)
VREF     = 3.3      # ADC reference voltage

def read_temperature():
    raw     = ntc.read_u16()
    voltage = raw * VREF / 65535        # Voltage at ADC pin

    if voltage <= 0 or voltage >= VREF:
        return None                     # Guard against divide-by-zero

    # Calculate NTC resistance from voltage divider
    r_ntc = R_FIXED * voltage / (VREF - voltage)

    # Steinhart-Hart simplified (Beta) equation
    temp_k = 1.0 / (
        1.0 / (T_NOM + 273.15) +
        (1.0 / B_COEFF) * math.log(r_ntc / R_NOM)
    )
    temp_c = temp_k - 273.15
    return round(temp_c, 1)

while True:
    temp = read_temperature()
    if temp is not None:
        print("Temperature:", temp, "°C")
    else:
        print("Sensor read error")
    utime.sleep(1)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and an **NTC Thermistor** onto the canvas.
2. Connect the top of the fixed resistor to **3.3V**, the mid-point junction to **GP26**, and the bottom of the NTC to **GND**. In MbedO, the NTC component includes its pull-up resistor automatically.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Adjust the NTC temperature slider and watch the serial terminal update.

## Expected Output
```
Temperature: 24.3 °C
Temperature: 24.4 °C
Temperature: 31.7 °C
```

## Expected Canvas Behavior
- Sliding the NTC temperature slider up increases the printed temperature.
- Sliding it down decreases the printed temperature.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `raw * VREF / 65535` | Converts the 16-bit ADC reading to the actual voltage at the GP26 pin. |
| `R_FIXED * voltage / (VREF - voltage)` | Derives the current NTC resistance from the voltage divider mid-point voltage. |
| `math.log(r_ntc / R_NOM)` | Takes the natural log of the resistance ratio — a core part of the Steinhart-Hart equation. |
| `temp_k - 273.15` | Converts the result from Kelvin to Celsius. |

## Hardware & Safety Concept: Steinhart-Hart Equation
The Steinhart-Hart equation is an empirical model that accurately describes how NTC thermistor resistance varies with temperature. The simplified "Beta" form used here requires just two parameters: the nominal resistance at 25 °C and the Beta (B) coefficient — both provided on the thermistor datasheet. Using the raw ADC value directly as temperature would give wildly inaccurate results due to the non-linear relationship between NTC resistance and temperature.

## Try This! (Challenges)
1. **Fahrenheit Output**: Add a conversion (`temp_f = temp * 9/5 + 32`) and print both Celsius and Fahrenheit.
2. **Threshold Alert**: Flash an LED on GP15 if the temperature exceeds 30 °C.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Reads unrealistic temperatures (e.g. -200 °C) | Wrong B coefficient | Check the NTC datasheet for the correct B value and update `B_COEFF`. |
| Always reads 0 °C or very high | Resistor wired incorrectly | Ensure the 10 kΩ fixed resistor is on top (3.3V side) and NTC is on the bottom (GND side). |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**. The `math` module is included in the MbedO MicroPython runtime.

## Related Projects
- [10 - Pico LDR Light Sensor Serial](10-pico-ldr-serial.md)
- [11 - Pico Potentiometer Serial](11-pico-potentiometer-serial.md)
