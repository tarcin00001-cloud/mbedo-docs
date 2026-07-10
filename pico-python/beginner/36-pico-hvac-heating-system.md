# 36 - Pico HVAC Heating System

Build an automated home heating thermostat that turns ON a heater relay when it gets cold, and displays active states on status LEDs.

## Goal
Learn how to read NTC temperature sensors, calculate temperature differentials, and implement hysteresis logic to control a high-power heating relay and status indicators in MicroPython.

## What You Will Build
A digital heating thermostat:
- **NTC Thermistor (GP26)**: Measures room temperature.
- **Relay Module (GP15)**: Controls power to a simulated heater. It turns ON when temperature drops below 18 °C and OFF when temperature rises above 20 °C.
- **Red LED (GP13)**: Turns ON when heating is active.
- **Green LED (GP14)**: Turns ON when the heating is on standby (temperature is safe).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| NTC Thermistor | `ntc` | Yes | Yes (10k NTC) |
| Relay Module | `relay` | Yes | Yes (controls heater load) |
| Red, Green LEDs | `led` | Yes | Yes (status indicators) |
| 10 kΩ Resistor | `resistor` | Optional in MbedO | Yes (voltage divider) |
| 330 Ω Resistors × 2 | `resistor` | Optional in MbedO | Yes — one per LED |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| 10 kΩ Resistor | Pin 1 | 3.3V (3V3) | Red | Top of voltage divider |
| 10 kΩ Resistor | Pin 2 | GP26 | Yellow | Mid-point junction |
| NTC Thermistor | Pin 1 | GP26 | Yellow | Analog input pin |
| NTC Thermistor | Pin 2 | GND | Black | Shared return path to ground |
| Relay Module | IN | GP15 | Orange | Heater relay signal |
| LED Red | Anode (+, longer leg) | GP13 | Red | Heating active indicator |
| LED Green | Anode (+, longer leg) | GP14 | Green | Heating standby indicator |
| All Cathodes | Cathode (−) | GND | Black | Shared return path to ground |
| 330 Ω Resistors | Either leg | In series with LED Anodes | — | Limits LED current |

> **Wiring tip:** Connect the fixed 10 kΩ resistor between 3.3V and GP26, and the NTC thermistor between GP26 and GND. Connect the relay input to GP15, and the LEDs to GP13 and GP14.

## Code
```python
from machine import Pin, ADC
import utime
import math

ntc = ADC(26) # GP26 = ADC channel 0
relay = Pin(15, Pin.OUT)

led_heat = Pin(13, Pin.OUT)
led_safe = Pin(14, Pin.OUT)

# Config constants
R_FIXED = 10000
R_NOM   = 10000
T_NOM   = 25.0
B_COEFF = 3950
VREF    = 3.3

# Hysteresis heating thresholds
TEMP_LOW_LIMIT  = 18.0 # Turn heater ON if temp < 18 °C
TEMP_HIGH_LIMIT = 20.0 # Turn heater OFF if temp > 20 °C

relay.value(0) # Start OFF
led_heat.value(0)
led_safe.value(1) # Start in safe state
heat_state = False

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
        print("Temp:", round(temp, 1), "C | Heater:", "ON" if heat_state else "OFF")
        
        # Hysteresis logic
        if temp < TEMP_LOW_LIMIT and not heat_state:
            heat_state = True
            relay.value(1)    # Turn heater ON
            led_heat.value(1) # Turn heating active LED ON
            led_safe.value(0) # Turn standby LED OFF
            print(">> Heating Active - Heater ON")
        elif temp > TEMP_HIGH_LIMIT and heat_state:
            heat_state = False
            relay.value(0)    # Turn heater OFF
            led_heat.value(0) # Turn heating active LED OFF
            led_safe.value(1) # Turn standby LED ON
            print(">> Target Temp Reached - Heater OFF")
            
    utime.sleep(1) # 1-second update cycle
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **NTC Thermistor**, **Relay**, and **two LEDs** (Red, Green) onto the canvas.
2. Connect NTC to **GP26**, Relay to **GP15**, Red LED to **GP13**, and Green LED to **GP14**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Adjust the NTC temperature slider below 18 °C to see the relay activate, then slide it up above 20 °C to see it turn OFF.

## Expected Output
```
Temp: 22.4 C | Heater: OFF
Temp: 17.5 C | Heater: OFF
>> Heating Active - Heater ON
Temp: 19.2 C | Heater: ON
Temp: 20.3 C | Heater: ON
>> Target Temp Reached - Heater OFF
```

## Expected Canvas Behavior
- The relay clicks active, the Red LED turns ON, and the Green LED turns OFF when the temperature drops below 18 °C.
- The Green LED turns ON and the Red LED turns OFF when the temperature rises above 20 °C.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `temp < TEMP_LOW_LIMIT` | Triggers the heating system to turn ON if the temperature falls below the low limit. |
| `relay.value(0)` | Deactivates the heater relay when the high limit is reached. |

## Hardware & Safety Concept: Heater Overheat Cut-offs
Heating systems must include thermal safety features to prevent fires or overheating in the event of software failure. Always place a secondary, hardware-based thermal fuse in series with the heater's power line. This fuse will physically break the circuit if the temperature exceeds a safety limit, protecting the system from thermal runaway.

## Try This! (Challenges)
1. **Critical High Alarm**: Add a buzzer on GP12 that sounds an alarm beep if the temperature exceeds 35 °C.
2. **Dynamic Backlight**: Turn off the status LEDs automatically if no state changes occur for 10 seconds to save energy.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Heater cycles ON/OFF rapidly | Hysteresis range too narrow | Ensure there is a sufficient gap (at least 1.5 - 2 °C) between your high and low thresholds. |
| Heater never turns ON | Incorrect voltage calculations | Print the raw ADC values to check the sensor scaling and verify your wiring. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [12 - Pico NTC Thermistor Serial](12-pico-ntc-serial.md)
- [24 - Pico Temperature Alarm NTC](24-pico-temperature-alarm-ntc.md)
- [35 - Pico Temperature Cooling Fan](35-pico-temperature-cooling-fan.md)
