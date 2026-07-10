# 31 - Pico Temperature Range Indicator

Display comfort zone indicators (Cold, Comfort, Hot) across three separate LEDs based on NTC temperature sensor readings.

## Goal
Learn how to read an NTC thermistor, compute Celsius values, and implement multi-stage comfort zone threshold checks to control three digital output channels in MicroPython.

## What You Will Build
A visual comfort index monitor:
- **NTC Thermistor (GP26)**: Measures room temperature.
- **Red LED (GP13)**: Lights up when hot (> 30 °C).
- **Yellow LED (GP14)**: Lights up when cold (< 18 °C).
- **Green LED (GP15)**: Lights up when in the comfort zone (18 °C to 30 °C).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| NTC Thermistor | `ntc` | Yes | Yes (10k NTC) |
| Red, Yellow, Green LEDs | `led` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | Optional in MbedO | Yes (voltage divider) |
| 330 Ω Resistors × 3 | `resistor` | Optional in MbedO | Yes — one per LED |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| 10 kΩ Resistor | Pin 1 | 3.3V (3V3) | Red | Top of voltage divider |
| 10 kΩ Resistor | Pin 2 | GP26 | Yellow | Mid-point junction |
| NTC Thermistor | Pin 1 | GP26 | Yellow | Analog input pin |
| NTC Thermistor | Pin 2 | GND | Black | Shared return path to ground |
| LED Red | Anode (+, longer leg) | GP13 | Red | Hot indicator |
| LED Yellow | Anode (+, longer leg) | GP14 | Yellow | Cold indicator |
| LED Green | Anode (+, longer leg) | GP15 | Green | Comfort indicator |
| All Cathodes | Cathode (−) | GND | Black | Shared return path to ground |
| 330 Ω Resistors | Either leg | In series with LED Anodes | — | Limits LED current |

> **Wiring tip:** Connect the fixed 10 kΩ resistor between 3.3V and GP26, and the NTC thermistor between GP26 and GND. Connect the three LED anodes to GP13, GP14, and GP15. All cathodes connect to ground.

## Code
```python
from machine import Pin, ADC
import utime
import math

ntc = ADC(26) # GP26 = ADC channel 0

led_hot  = Pin(13, Pin.OUT)
led_cold = Pin(14, Pin.OUT)
led_comf = Pin(15, Pin.OUT)

# Config constants
R_FIXED = 10000
R_NOM   = 10000
T_NOM   = 25.0
B_COEFF = 3950
VREF    = 3.3

# Temperature limits
LIMIT_COLD = 18.0
LIMIT_HOT  = 30.0

def get_temp():
    raw = ntc.read_u16()
    voltage = raw * VREF / 65535
    if voltage <= 0 or voltage >= VREF:
        return None
    r_ntc = R_FIXED * voltage / (VREF - voltage)
    temp_k = 1.0 / (1.0 / (T_NOM + 273.15) + (1.0 / B_COEFF) * math.log(r_ntc / R_NOM))
    return temp_k - 273.15

def set_indicators(h, c, ok):
    led_hot.value(h)
    led_cold.value(c)
    led_comf.value(ok)

while True:
    temp = get_temp()
    
    if temp is not None:
        print("Temp:", round(temp, 1), "C")
        
        # Comfort zone threshold logic
        if temp < LIMIT_COLD:
            set_indicators(0, 1, 0) # Cold (Yellow ON)
            print(">> Zone status: COLD")
        elif temp > LIMIT_HOT:
            set_indicators(1, 0, 0) # Hot (Red ON)
            print(">> Zone status: HOT")
        else:
            set_indicators(0, 0, 1) # Comfort (Green ON)
            print(">> Zone status: COMFORT")
            
    utime.sleep(1) # 1-second update cycle
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **NTC Thermistor**, and **three LEDs** (Red, Yellow, Green) onto the canvas.
2. Connect NTC to **GP26**. Connect Red to **GP13**, Yellow to **GP14**, Green to **GP15**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Adjust the NTC temperature slider and watch the LEDs cycle.

## Expected Output
```
Temp: 16.5 C
>> Zone status: COLD
Temp: 24.3 C
>> Zone status: COMFORT
Temp: 32.1 C
>> Zone status: HOT
```

## Expected Canvas Behavior
- The Green LED glows at normal temperatures.
- Sliding the NTC slider down turns ON the Yellow LED.
- Sliding the NTC slider up turns ON the Red LED.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `temp < LIMIT_COLD` | Checks if the measured temperature falls below the cold comfort boundary. |
| `set_indicators(0, 0, 1)` | Activates only the Green comfort LED when within target limits. |

## Hardware & Safety Concept: Multi-State Status Displays
Comfort zone indicators are commonly found on home air monitors, storage spaces, and cleanroom displays. Using simple color-coded status LEDs allows users to monitor status at a glance without reading text on a display screen.

## Try This! (Challenges)
1. **Auditory Alarm**: Connect a buzzer on GP12 and sound a beep if the temperature enters the Hot or Cold alert zones.
2. **Hysteresis boundaries**: Implement a small hysteresis band (e.g. 0.5 °C) to prevent indicators from flickering at transition boundaries.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Indicators display incorrect states | Pins mapped out of order | Check that GP13 = Red (Hot), GP14 = Yellow (Cold), and GP15 = Green (Comfort). |
| Temperature reads extreme defaults | Voltage divider wiring issue | Ensure the fixed 10 kΩ resistor is on the 3.3V side, and NTC is on the GND side. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [12 - Pico NTC Thermistor Serial](12-pico-ntc-serial.md)
- [24 - Pico Temperature Alarm NTC](24-pico-temperature-alarm-ntc.md)
- [25 - Pico HVAC Control](25-pico-hvac-control.md)
