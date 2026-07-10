# 35 - Pico Temperature Cooling Fan

Build a smart cooling fan system that switches ON a DC motor fan when the temperature rises above comfort limits.

## Goal
Learn how to read an NTC thermistor, compute Celsius values, and drive a DC motor driver interface (Relay/Transistor) to control a cooling fan in MicroPython.

## What You Will Build
A temperature-controlled cooling fan:
- **NTC Thermistor (GP26)**: Measures room temperature.
- **Relay Module (GP15)**: Switches power ON to a DC motor fan when the temperature exceeds 28 °C, and OFF when it drops below 26 °C.
- **LED (GP14)**: Flashes when the fan is active to provide visual status feedback.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| NTC Thermistor | `ntc` | Yes | Yes (10k NTC) |
| Relay Module | `relay` | Yes | Yes (controls DC motor) |
| DC Motor (Fan) | `dc_motor` | Yes | Yes |
| LED (any colour) | `led` | Yes | Yes (status indicator) |
| 10 kΩ Resistor | `resistor` | Optional in MbedO | Yes (voltage divider) |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| 10 kΩ Resistor | Pin 1 | 3.3V (3V3) | Red | Top of voltage divider |
| 10 kΩ Resistor | Pin 2 | GP26 | Yellow | Mid-point junction |
| NTC Thermistor | Pin 1 | GP26 | Yellow | Analog input pin |
| NTC Thermistor | Pin 2 | GND | Black | Shared return path to ground |
| Relay Module | IN | GP15 | Orange | Cooling fan relay signal |
| LED | Anode (+, longer leg) | GP14 | Yellow | Fan active indicator |
| LED | Cathode (−, shorter leg) | GND | Black | Shared return path to ground |
| 330 Ω Resistor | Either leg | In series with LED Anode | — | Limits LED current |
| DC Motor | VCC / GND | Relay COM / GND | — | Motor power circuit |

> **Wiring tip:** Connect the fixed 10 kΩ resistor between 3.3V and GP26, and the NTC thermistor between GP26 and GND. Connect the relay input pin to GP15, and the LED anode to GP14.

## Code
```python
from machine import Pin, ADC
import utime
import math

ntc = ADC(26) # GP26 = ADC channel 0
relay = Pin(15, Pin.OUT)
led   = Pin(14, Pin.OUT)

# Config constants
R_FIXED = 10000
R_NOM   = 10000
T_NOM   = 25.0
B_COEFF = 3950
VREF    = 3.3

# Hysteresis limits
TEMP_HIGH_LIMIT = 28.0 # Turn fan ON if temp > 28 °C
TEMP_LOW_LIMIT  = 26.0 # Turn fan OFF if temp < 26 °C

relay.value(0) # Start OFF
led.value(0)
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
        
        # Hysteresis logic
        if temp > TEMP_HIGH_LIMIT and not fan_state:
            fan_state = True
            relay.value(1) # Turn cooling fan ON
            led.value(1)   # Turn status LED ON
            print(">> Cooling Fan Activated")
        elif temp < TEMP_LOW_LIMIT and fan_state:
            fan_state = False
            relay.value(0) # Turn cooling fan OFF
            led.value(0)   # Turn status LED OFF
            print(">> Cooling Fan Deactivated")
            
    utime.sleep(1) # 1-second update cycle
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **NTC Thermistor**, **Relay**, **DC Motor**, and **LED** onto the canvas.
2. Connect NTC to **GP26**, Relay to **GP15**, and LED to **GP14**. Connect DC Motor terminals to the relay.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Adjust the NTC temperature slider above 28 °C to see the motor spin, then slide it down below 26 °C to see it stop.

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
- The relay clicks active, the LED turns ON, and the DC motor spins when the temperature exceeds 28 °C.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `relay.value(1)` | Drives GP15 HIGH to activate the relay, completing the motor power circuit. |
| `temp < TEMP_LOW_LIMIT` | Checks if the temperature has dropped below the low limit to deactivate the fan. |

## Hardware & Safety Concept: Inductive Load Back-EMF Protection
DC motors are inductive loads that generate high-voltage spikes (back-EMF) when turned OFF. When driving DC motors using transistors or relays, always add a **flyback diode** (e.g. 1N4007) in reverse parallel across the motor terminals. This diode provides a safe path for the current to dissipate, protecting the relay contacts and microcontroller from damage.

## Try This! (Challenges)
1. **Pulse Width Modulation Speed**: Connect the motor driver pin to a PWM channel and run the motor at half speed when moderate, and full speed when hot.
2. **Dynamic Indicator**: Flash the status LED at different rates depending on the fan speed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor hums but does not spin | Power supply current limit | DC motors require high startup current. Ensure your power supply can provide enough current to drive the motor load. |
| Relay clicks but motor stays OFF | Motor power circuit open | Check that the motor's power supply is wired correctly through the relay's Normally Open (NO) and Common (COM) terminals. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [12 - Pico NTC Thermistor Serial](12-pico-ntc-serial.md)
- [24 - Pico Temperature Alarm NTC](24-pico-temperature-alarm-ntc.md)
- [25 - Pico HVAC Control](25-pico-hvac-control.md)
