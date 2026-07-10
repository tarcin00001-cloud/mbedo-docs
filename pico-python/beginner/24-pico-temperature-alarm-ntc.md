# 24 - Pico Temperature Alarm NTC

Build a high-temperature threshold safety alarm that sounds a warning buzzer when temperature exceeds comfort bounds.

## Goal
Learn how to read an NTC thermistor, compute Celsius values, and trigger a warning alarm using digital output controls in MicroPython.

## What You Will Build
A temperature warning monitor:
- **NTC Thermistor (GP26)**: Measures temperature.
- **Active Buzzer (GP15)**: Emits warning beeps if the temperature rises above 30 °C.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| NTC Thermistor | `ntc` | Yes | Yes (10k NTC) |
| Active Buzzer | `buzzer` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | Optional in MbedO | Yes (voltage divider) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| 10 kΩ Resistor | Pin 1 | 3.3V (3V3) | Red | Top of voltage divider |
| 10 kΩ Resistor | Pin 2 | GP26 | Yellow | Mid-point junction |
| NTC Thermistor | Pin 1 | GP26 | Yellow | Analog input pin |
| NTC Thermistor | Pin 2 | GND | Black | Shared return path to ground |
| Active Buzzer | VCC (+) | GP15 | Orange | Warning buzzer signal |
| Active Buzzer | GND (−) | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the fixed 10 kΩ resistor between 3.3V and GP26, and the NTC thermistor between GP26 and GND. Connect the buzzer to GP15 and GND.

## Code
```python
from machine import Pin, ADC
import utime
import math

ntc = ADC(26) # GP26 = ADC channel 0
buzzer = Pin(15, Pin.OUT)

# Config constants
R_FIXED = 10000
R_NOM   = 10000
T_NOM   = 25.0
B_COEFF = 3950
VREF    = 3.3

# Temperature limit in Celsius
TEMP_LIMIT = 30.0

buzzer.value(0) # Start silent

def get_temp():
    raw = ntc.read_u16()
    voltage = raw * VREF / 65535
    if voltage <= 0 or voltage >= VREF:
        return None
    r_ntc = R_FIXED * voltage / (VREF - voltage)
    temp_k = 1.0 / (1.0 / (T_NOM + 273.15) + (1.0 / B_COEFF) * math.log(r_ntc / R_NOM))
    return temp_k - 273.15

while True:
    temp = get_temp()
    
    if temp is not None:
        print("Temp:", round(temp, 1), "C")
        
        # Check alarm threshold
        if temp > TEMP_LIMIT:
            print("ALERT: Overheating!")
            # Pulse buzzer
            buzzer.value(1)
            utime.sleep_ms(150)
            buzzer.value(0)
            utime.sleep_ms(150)
        else:
            buzzer.value(0)
            utime.sleep_ms(300)
    else:
        utime.sleep_ms(300)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **NTC Thermistor**, and **Active Buzzer** onto the canvas.
2. Connect NTC pull-up to **3.3V** and signal to **GP26**. Connect buzzer to **GP15**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the NTC temperature slider above 30 °C to trigger the alarm.

## Expected Output
```
Temp: 24.5 C
Temp: 31.2 C
ALERT: Overheating!
```

## Expected Canvas Behavior
- The buzzer component pulses active when the NTC temperature slider is moved above 30 °C.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `temp > TEMP_LIMIT` | Checks if the computed temperature exceeds the safety threshold. |
| `buzzer.value(0)` | Silences the alarm if the temperature returns below the threshold limit. |

## Hardware & Safety Concept: Overheating Protection
Overheating safety monitors are critical for protecting engines, power electronics, and servers. Using a reliable thermal sensor to trigger warning chimes allows operators or automated systems to shut down equipment safely before thermal damage occurs.

## Try This! (Challenges)
1. **Critical Cooling Fan**: Add an LED/Relay on GP14 to simulate a cooling fan that turns ON when the alarm is active.
2. **Freeze Warning**: Add a low-temperature threshold alert at 5 °C (for frost protection).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Temperature value stays constant | Wiring error in voltage divider | Check that the NTC thermistor and fixed resistor are configured correctly at GP26. |
| Alarm triggers at normal room temperature | Threshold set too low | Verify the ambient room temperature reading and adjust `TEMP_LIMIT` accordingly. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [12 - Pico NTC Thermistor Serial](12-pico-ntc-serial.md)
- [25 - Pico HVAC Control](25-pico-hvac-control.md)
