# 186 - Pico MicroPython Custom Module

Build an organized HVAC monitoring console that structures its NTC thermistor conversion and hysteresis calculations inside a custom, reusable MicroPython library.

## Goal
Learn how to create custom MicroPython library files (`.py` modules), upload them to the Pico's flash filesystem, import them using the standard `import` statement, and organize code modularly.

## What You Will Build
A modular climate controller:
- **Custom Module (`hvac_utils.py`)**: A separate file containing calculations for NTC thermistors and dual-limit hysteresis triggers.
- **Main Script (`main.py`)**: Imports `hvac_utils` to read temperature from GP26 and toggle a Heater Relay (GP10).
- **16x2 I2C LCD (GP4, GP5)**: Displays temperature and target parameters.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| NTC Thermistor | `potentiometer` | Yes (represented by potentiometer) | Yes (10k NTC) |
| Relay Module | `relay` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 10k-ohm Resistors | `resistor` | Optional | Yes (voltage divider) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NTC Thermistor | Signal (AO) | GP26 | Yellow | Analog temperature input |
| NTC Thermistor | VCC / GND | 3.3V / GND | Red / Black | Power lines |
| Relay Module | IN | GP10 | Orange | Heater control switch |
| Relay Module | VCC / GND | 5V / GND | Red / Black | Power lines |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the NTC thermistor output to GP26. The Relay Module connects to GP10. The LCD uses shared pins GP4/GP5.

## Code

### Library File: [hvac_utils.py](file:///c:/Users/LocalAdmin/working/Dev/MbedO/mbedo/docs/projects/pico-python/expert/hvac_utils.py)
Save this code in a file named `hvac_utils.py` on the Pico's root directory:
```python
# hvac_utils.py
# Custom MicroPython Module for HVAC Calculations

def raw_to_temp(raw_val):
    """Converts a raw 16-bit NTC reading (0-65535) to Celsius."""
    # Approximate linear temperature scaling around 25C
    return 25.0 + (32768 - raw_val) * 0.0025

def check_hysteresis(current_temp, setpoint, band, current_state):
    """
    Evaluates whether the heating system should turn ON or OFF.
    Returns: True if heater should be ON, False if OFF.
    """
    if current_state:
        # If heater is currently ON, turn it OFF if we exceed setpoint + band
        if current_temp > (setpoint + band):
            return False
        return True
    else:
        # If heater is currently OFF, turn it ON if we fall below setpoint - band
        if current_temp < (setpoint - band):
            return True
        return False
```

### Main Application File: [main.py](file:///c:/Users/LocalAdmin/working/Dev/MbedO/mbedo/docs/projects/pico-python/expert/main.py)
Save this code in a file named `main.py` on the Pico's root directory:
```python
# main.py
from machine import Pin, ADC, I2C
import utime
from machine_lcd import I2cLcd

# Import custom library
import hvac_utils

therm = ADC(26)
relay = Pin(10, Pin.OUT)
relay.value(0)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Settings
TARGET_SETPOINT = 24.0
HYSTERESIS_BAND = 1.5

heater_on = False

lcd.clear()
lcd.putstr("Modular Therm\nLibrary Loaded!")
utime.sleep(1.5)

print("Modular thermostat active.")

while True:
    raw_reading = therm.read_u16()
    
    # 1. Use the custom module to calculate temperature
    temp_c = hvac_utils.raw_to_temp(raw_reading)
    
    # 2. Use the custom module to calculate relay state
    heater_on = hvac_utils.check_hysteresis(
        temp_c, TARGET_SETPOINT, HYSTERESIS_BAND, heater_on
    )
    
    # Actuate relay
    relay.value(1 if heater_on else 0)
    
    # 3. Update LCD
    lcd.clear()
    lcd.putstr("Temp: {:.1f}C\n".format(temp_c))
    lcd.putstr("Set: {} Htr:{}".format(TARGET_SETPOINT, "ON" if heater_on else "OFF"))
    
    print("Temp: {:.1f}C | Setpoint: {}C | Htr: {}".format(
        temp_c, TARGET_SETPOINT, "ON" if heater_on else "OFF"
    ))
    
    utime.sleep_ms(500)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **NTC Thermistor**, **Relay**, and **I2C LCD** onto the canvas.
2. Connect Thermistor to **GP26**, Relay to **GP10**, and LCD to **GP4/GP5**.
3. Create a new file in MbedO workspace named `hvac_utils.py` and paste the library code into it.
4. Open the `main.py` file, paste the main application code, and click **Run**.
5. Adjust the NTC temperature slider. Watch the heater relay state change on the LCD, verifying that the imported library functions are running.

## Expected Output
Terminal:
```
Modular thermostat active.
Temp: 24.5C | Setpoint: 24.0C | Htr: OFF
Temp: 22.1C | Setpoint: 24.0C | Htr: ON
```

## Expected Canvas Behavior
* Startup: LCD reads `Modular Therm` / `Library Loaded!` for 1.5 seconds.
* Temperature falls below 22.5°C: Relay turns ON. LCD shows `Htr:ON`.
* Temperature rises above 25.5°C: Relay turns OFF. LCD shows `Htr:OFF`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `import hvac_utils` | Tells the MicroPython parser to search the flash memory filesystem for `hvac_utils.py` and load its symbols. |
| `hvac_utils.raw_to_temp(...)` | Calls the `raw_to_temp` function exported by the custom `hvac_utils` library namespace. |

## Hardware & Safety Concept: Code Modularization and Unit Testing
In complex firmware projects, putting all logic into a single large file makes debugging difficult. Structuring conversion math, parsing algorithms, and networking protocols into independent modules allows engineers to write unit tests for each module separately. For example, the `hvac_utils.py` code can be tested on a computer using standard Python before uploading it to run on the Pico.

## Try This! (Challenges)
1. **Celsius to Fahrenheit**: Add a function `c_to_f(temp_c)` to the `hvac_utils.py` library and modify `main.py` to display the temperature in Fahrenheit.
2. **Alert Function**: Add a function `check_alert(temp_c, threshold)` to the module that returns `True` if the temperature exceeds a safety limit, and wire GP15 to beep a buzzer if active.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| `ImportError: no module named 'hvac_utils'` | File not saved in root | Ensure the file is named exactly `hvac_utils.py` (case sensitive) and is saved in the same directory/folder as `main.py` on the Pico. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [105 - Pico Thermostat](../intermediate/105-pico-thermostat.md)
- [169 - Pico Smart Thermostat EEPROM](../advanced/169-pico-smart-thermostat-eeprom.md)
