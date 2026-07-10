# 25 - Pico HVAC Control

Build an HVAC comfort controller that turns ON a cooling fan (LED/Relay) when it gets hot, and OFF when it cools down.

## Goal
Learn how to implement dual-threshold comfort range checking using an NTC thermistor to control an HVAC cooling actuator in MicroPython.

## What You Will Build
An HVAC climate switch:
- **NTC Thermistor (GP26)**: Measures room temperature.
- **LED/Relay (GP15)**: Simulates a cooling fan. It turns ON when the temperature exceeds 28 °C and stays ON until the temperature drops below 26 °C (implementing hysteresis to prevent rapid cycling).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| NTC Thermistor | `ntc` | Yes | Yes (10k NTC) |
| LED (any colour) | `led` | Yes | Yes (simulates cooling fan) |
| 10 kΩ Resistor | `resistor` | Optional in MbedO | Yes (voltage divider) |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| 10 kΩ Resistor | Pin 1 | 3.3V (3V3) | Red | Top of voltage divider |
| 10 kΩ Resistor | Pin 2 | GP26 | Yellow | Mid-point junction |
| NTC Thermistor | Pin 1 | GP26 | Yellow | Analog input pin |
| NTC Thermistor | Pin 2 | GND | Black | Shared return path to ground |
| LED | Anode (+, longer leg) | GP15 | Orange | Cooling fan relay signal |
| LED | Cathode (−, shorter leg) | GND | Black | Shared return path to ground |
| 330 Ω Resistor | Either leg | In series with LED Anode | — | Limits LED current |

> **Wiring tip:** Connect the fixed 10 kΩ resistor between 3.3V and GP26, and the NTC thermistor between GP26 and GND. Connect the LED anode to GP15 through the current-limiting resistor.

## Code
```python
from machine import Pin, ADC
import utime
import math

ntc = ADC(26) # GP26 = ADC channel 0
fan = Pin(15, Pin.OUT)

# Config constants
R_FIXED = 10000
R_NOM   = 10000
T_NOM   = 25.0
B_COEFF = 3950
VREF    = 3.3

# Hysteresis comfort thresholds
TEMP_HIGH_LIMIT = 28.0 # Turn fan ON if temp > 28 °C
TEMP_LOW_LIMIT  = 26.0 # Turn fan OFF if temp < 26 °C

fan.value(0) # Start OFF
fan_state = False

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
        print("Temp:", round(temp, 1), "C | Fan:", "ON" if fan_state else "OFF")
        
        # Dual-threshold hysteresis logic
        if temp > TEMP_HIGH_LIMIT and not fan_state:
            fan_state = True
            fan.value(1) # Turn cooling fan ON
            print(">> Cooling Fan Activated")
        elif temp < TEMP_LOW_LIMIT and fan_state:
            fan_state = False
            fan.value(0) # Turn cooling fan OFF
            print(">> Cooling Fan Deactivated")
            
    utime.sleep(1) # 1-second update cycle
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **NTC Thermistor**, and **LED** onto the canvas.
2. Connect NTC pull-up to **3.3V** and signal to **GP26**. Connect LED to **GP15**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Adjust the NTC temperature slider above 28 °C to see the LED turn ON, then slide it down below 26 °C to see it turn OFF.

## Expected Output
```
Temp: 25.4 C | Fan: OFF
Temp: 28.5 C | Fan: OFF
>> Cooling Fan Activated
Temp: 27.2 C | Fan: ON
Temp: 25.8 C | Fan: ON
>> Cooling Fan Deactivated
```

## Expected Canvas Behavior
- The LED component on the canvas lights up when the temperature rises past 28 °C and turns OFF only after dropping back below 26 °C.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `temp > TEMP_HIGH_LIMIT` | Triggers the cooling fan to turn ON if the temperature exceeds the high comfort limit. |
| `temp < TEMP_LOW_LIMIT` | Turns the cooling fan OFF only after the temperature falls below the low safety limit. |
| `fan_state` | Boolean flag that tracks the active state of the fan, preventing redundant actions and print statements. |

## Hardware & Safety Concept: HVAC Hysteresis
If an HVAC controller used a single threshold (e.g. turning the fan ON at 27.0 °C and OFF at 26.9 °C), minor sensor noise would cause the fan relay to switch ON and OFF rapidly (chattering). This rapid cycling causes wear on motors and welds relay contacts. Using separate high and low limits (hysteresis) prevents chattering.

## Try This! (Challenges)
1. **Heat Mode**: Invert the logic so the relay controls a heater: turn ON when cold (< 18 °C) and OFF when warm (> 20 °C).
2. **Speed-Controlled Fan**: Connect the fan pin to a PWM channel and run the fan at different speeds based on the temperature.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Fan cycles ON/OFF rapidly | Hysteresis range too narrow | Ensure there is a sufficient gap (at least 1.5 - 2 °C) between your high and low thresholds. |
| Fan never turns ON | Incorrect voltage calculations | Print the raw ADC values to check the sensor scaling and verify your wiring. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [12 - Pico NTC Thermistor Serial](12-pico-ntc-serial.md)
- [24 - Pico Temperature Alarm NTC](24-pico-temperature-alarm-ntc.md)
