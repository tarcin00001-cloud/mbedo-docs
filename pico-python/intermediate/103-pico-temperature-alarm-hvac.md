# 103 - Pico Temperature Alarm HVAC

Build a temperature threshold safety alarm that flashes a warning LED and sounds a pulsing warning buzzer when ambient temperatures exceed safe operating limits.

## Goal
Learn how to read analog inputs from an NTC thermistor, calculate temperatures using standard formulas, implement threshold alarm logic, and control warning LEDs and buzzers in MicroPython.

## What You Will Build
A temperature safety alarm:
- **NTC Thermistor (GP26)**: Measures temperature.
- **LED (GP14)**: Flashes during alarm states.
- **Active Buzzer (GP15)**: Sounds a pulsing warning siren when triggered.
- **Reset Button (GP13)**: Silences the alarm once the temperature returns to safe limits.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| NTC Thermistor | `thermistor` | Yes | Yes |
| LED (any colour) | `led` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Tactile Push Button | `button` | Yes (reset switch) | Yes |
| 10 kΩ Resistor | `resistor` | Optional in MbedO | Yes (voltage divider) |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Thermistor | Pin 1 | 3.3V (3V3) | Red | Power line |
| Thermistor | Pin 2 | GP26 | Yellow | Analog input signal |
| 10 kΩ Resistor | Either leg | GP26 to GND | Black | Pull-down resistor |
| Reset Button | Terminal 1 | GP13 | White | Reset input (reads LOW on press) |
| Reset Button | Terminal 2 | GND | Black | Shorts GP13 to GND |
| LED | Anode (+, longer leg) | GP14 | Orange | Warning indicator output pin |
| LED | Cathode (−, shorter leg) | GND | Black | Shared return path to ground |
| 330 Ω Resistor | Either leg | In series with LED Anode | — | Limits LED current |
| Active Buzzer | VCC (+) | GP15 | Red | Alarm sound output pin |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The thermistor and 10 kΩ resistor form a voltage divider connected to GP26. Connect the reset button to GP13, LED anode to GP14, and active buzzer VCC to GP15. All grounds are shared.

## Code
```python
from machine import Pin, ADC
import utime
import math

thermistor = ADC(26) # GP26 = ADC channel 0
btn_reset  = Pin(13, Pin.IN, Pin.PULL_UP)
warning_l  = Pin(14, Pin.OUT)
buzzer     = Pin(15, Pin.OUT)

# Threshold: alarm triggers if temperature exceeds this limit (Celsius)
TEMP_LIMIT_C = 35.0

alarm_state = False
warning_l.value(0)
buzzer.value(0)

# Steinhart-Hart equation constants for 10k thermistor
B_COEFFICIENT = 3950
SERIES_RESISTOR = 10000
THERMISTOR_NOMINAL = 10000
TEMPERATURE_NOMINAL = 25

def get_temperature():
    raw = thermistor.read_u16()
    if raw == 0 or raw == 65535:
        return 0.0
    # Calculate resistance
    resistance = SERIES_RESISTOR / (65535 / raw - 1)
    
    # Calculate temperature in Celsius using B-parameter equation
    steinhart = resistance / THERMISTOR_NOMINAL
    steinhart = math.log(steinhart)
    steinhart /= B_COEFFICIENT
    steinhart += 1.0 / (TEMPERATURE_NOMINAL + 273.15)
    steinhart = 1.0 / steinhart
    temp_c = steinhart - 273.15
    return temp_c

print("Temperature safety alarm active.")

while True:
    temp_c = get_temperature()
    reset_pressed = (btn_reset.value() == 0)
    
    # Check for trigger (Set)
    if temp_c > TEMP_LIMIT_C and not alarm_state:
        alarm_state = True
        print(">> WARNING: High temperature detected! Alarm active. Temp: {:.1f} C".format(temp_c))
        
    # Check for disarm (Reset)
    if reset_pressed and alarm_state:
        alarm_state = False
        warning_l.value(0)
        buzzer.value(0)
        print(">> Temperature alarm reset.")
        utime.sleep(1) # Lockout delay
        
    # Pulsing warning siren
    if alarm_state:
        warning_l.value(1)
        buzzer.value(1)
        utime.sleep_ms(200)
        warning_l.value(0)
        buzzer.value(0)
        utime.sleep_ms(200)
    else:
        utime.sleep(1) # Normal polling speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Thermistor**, **Push Button** (reset switch), **LED**, and **Active Buzzer** onto the canvas.
2. Connect Thermistor Pin 1 to **3.3V** and Pin 2 to **GP26**.
3. Connect Reset Button to **GP13**, LED to **GP14**, and Buzzer to **GP15**. Connect all grounds.
4. Paste the code, select **MicroPython** mode, and click **Run**.
5. Slide the thermistor temperature slider past 35°C to trigger the alarm. Click Reset to disarm.

## Expected Output
```
Temperature safety alarm active.
>> WARNING: High temperature detected! Alarm active. Temp: 36.4 C
>> Temperature alarm reset.
```

## Expected Canvas Behavior
- The LED and buzzer components on the canvas pulse ON and OFF rapidly when the thermistor temperature slider is moved past 35°C.
- Clicking the Reset Button silences the alarm and turns the LED OFF.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `temp_c > TEMP_LIMIT_C` | Checks if the measured temperature is above the safety threshold, indicating overheating. |
| `alarm_state = True` | Latches the alarm state to `True` so the alarm continues to sound even if the temperature drops. |

## Hardware & Safety Concept: Alarm Latching and Reset Loops
Safety standards require **alarm latching** for critical parameters. Once a threshold is crossed, the alarm must continue to sound even if the parameter returns to safe limits (e.g. the room cools down again). This ensures that temporary hazard events are noticed by operators, and the system can only be silenced by an authorized manual reset.

## Try This! (Challenges)
1. **Visual strobe**: Connect a Red LED on GP12 and flash it in sync with the buzzer siren.
2. **Cooling Fan Relay**: Connect an LED/Relay on GP12 that turns ON (simulating a cooling fan) when the temperature exceeds the limit.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly | Threshold set too low | Print temperature values to the terminal to find the ambient level, then adjust `TEMP_LIMIT_C` in your code. |
| Alarm does not disarm | Reset button pin mismatch | Verify the reset button is wired to GP13 as configured in the code. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [12 - Pico NTC Thermistor Serial](12-pico-thermistor-serial.md)
- [31 - Pico Temperature Range Indicator](31-pico-temperature-range-indicator.md)
- [32 - Pico Temperature Buzzer Alarm](32-pico-temperature-buzzer-alarm.md)
