# 185 - Pico MicroPython Flash Config Parser

Build an intelligent climate controller that stores system configurations in a JSON file in flash, parses the settings on boot, and dynamically updates parameters.

## Goal
Learn how to use MicroPython's built-in `ujson` library to serialize and deserialize structured configuration files, save user settings, and load parameters at boot.

## What You Will Build
A config-driven thermostat station:
- **JSON Configuration File (`config.json`)**: Stores settings such as `setpoint` (default 25°C), `hysteresis` (default 2°C), and `device_name` (default "HVAC_PICO").
- **NTC Thermistor (GP26)**: Measures room temperature.
- **Relay Module (GP10)**: Toggles a heater based on the loaded JSON thresholds.
- **Button A (GP13)**: Increments the setpoint and writes it back to the JSON file.
- **Button B (GP14)**: Toggles the hysteresis value and saves it.
- **16x2 I2C LCD (GP4, GP5)**: Displays the device name, temperature, and setpoint.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| NTC Thermistor | `potentiometer` | Yes (represented by potentiometer) | Yes (10k NTC) |
| Relay Module | `relay` | Yes | Yes |
| Tactile Push Buttons × 2 | `button` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 10k-ohm Resistors | `resistor` | Optional | Yes (voltage divider) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NTC Thermistor | Signal (AO) | GP26 | Yellow | Analog temperature input |
| NTC Thermistor | VCC / GND | 3.3V / GND | Red / Black | Power lines |
| Relay Module | IN | GP10 | Orange | Heater control switch |
| Button A (Setpoint) | Terminal 1 | GP13 | White | Increase target setpoint |
| Button A (Setpoint) | Terminal 2 | GND | Black | Ground return |
| Button B (Hysteresis) | Terminal 1 | GP14 | Yellow | Toggle hysteresis value |
| Button B (Hysteresis) | Terminal 2 | GND | Black | Ground return |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the NTC thermistor to GP26. The Setpoint Button is on GP13 and the Hysteresis Button is on GP14. The LCD uses shared pins GP4/GP5.

## Code
```python
from machine import Pin, ADC, I2C
import utime, ujson, os
from machine_lcd import I2cLcd

# Hardware pins
therm = ADC(26)
relay = Pin(10, Pin.OUT)
relay.value(0)

btn_set = Pin(13, Pin.IN, Pin.PULL_UP)
btn_hyst = Pin(14, Pin.IN, Pin.PULL_UP)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

CONFIG_FILE = "config.json"

# Default configuration settings
default_config = {
    "setpoint": 24,
    "hysteresis": 2,
    "device_name": "HVAC_PICO"
}

def load_config():
    """Reads and parses the configuration file, creating a default one if missing."""
    try:
        with open(CONFIG_FILE, "r") as f:
            config = ujson.load(f)
            print("Configuration loaded successfully:", config)
            return config
    except Exception as e:
        print("Config file missing or corrupt. Creating default...")
        save_config(default_config)
        return default_config

def save_config(config):
    """Serialises and writes the configuration dictionary to flash in JSON format."""
    try:
        with open(CONFIG_FILE, "w") as f:
            ujson.dump(config, f)
        print("Configuration saved to flash:", config)
    except OSError as e:
        print("Failed to save config:", e)

# Load settings at boot
config = load_config()

lcd.clear()
lcd.putstr("Booting device\n{}".format(config["device_name"]))
utime.sleep(1.5)

last_btn = 0

while True:
    now = utime.ticks_ms()
    
    # Read temperature
    raw_val = therm.read_u16()
    temp_c = 25.0 + (32768 - raw_val) * 0.0025
    
    config_changed = False
    
    # 1. Handle Setpoint Button (GP13)
    # Cycles target setpoint between 20°C and 30°C
    if btn_set.value() == 0 and utime.ticks_diff(now, last_btn) > 400:
        last_btn = now
        config["setpoint"] += 1
        if config["setpoint"] > 30:
            config["setpoint"] = 20
        config_changed = True
        print("New setpoint target:", config["setpoint"])
        
    # 2. Handle Hysteresis Button (GP14)
    # Cycles hysteresis range between 1°C and 3°C
    if btn_hyst.value() == 0 and utime.ticks_diff(now, last_btn) > 400:
        last_btn = now
        config["hysteresis"] += 1
        if config["hysteresis"] > 3:
            config["hysteresis"] = 1
        config_changed = True
        print("New hysteresis band:", config["hysteresis"])
        
    # 3. Write updated configuration to flash
    if config_changed:
        save_config(config)
        
    # 4. HVAC Thermostat logic with hysteresis
    target = config["setpoint"]
    hyst = config["hysteresis"]
    
    if temp_c < (target - hyst):
        relay.value(1)  # Turn ON
    elif temp_c > (target + hyst):
        relay.value(0)  # Turn OFF
        
    # 5. Update LCD Screen
    lcd.clear()
    lcd.putstr("{}: {:.1f}C\n".format(config["device_name"], temp_c))
    lcd.putstr("SP:{} Hys:{} R:{}".format(target, hyst, "ON" if relay.value() else "OFF"))
    
    utime.sleep_ms(200)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **NTC Thermistor**, **Relay**, **two Buttons**, and **I2C LCD** onto the canvas.
2. Connect Thermistor to **GP26**, Relay to **GP10**, Buttons to **GP13/GP14**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Press Button A (GP13) to cycle setpoints. Press Button B (GP14) to toggle hysteresis. Restart the simulation. Verify that the new settings are automatically loaded on boot.

## Expected Output
Terminal:
```
Config file missing or corrupt. Creating default...
Configuration saved to flash: {'setpoint': 24, 'hysteresis': 2, 'device_name': 'HVAC_PICO'}
New setpoint target: 25
Configuration saved to flash: {'setpoint': 25, 'hysteresis': 2, 'device_name': 'HVAC_PICO'}
```

## Expected Canvas Behavior
* Boot: LCD reads `Booting device` / `HVAC_PICO` for 1.5 seconds.
* Press GP13: Setpoint (SP) increases, configuration is saved.
* Restart Simulation: LCD displays the newly saved setpoint (e.g. `SP:25`) immediately on boot.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `ujson.load(f)` | Decodes the JSON text file from the flash filesystem into a MicroPython dictionary. |
| `ujson.dump(config, f)` | Encodes the configuration dictionary into a JSON string and writes it to the file. |

## Hardware & Safety Concept: Config-Driven Embedded Firmware
Hardcoding values (like calibration coefficients, I2C addresses, or threshold limits) directly in compiled code makes firmware updates necessary every time a hardware parameter changes. Using JSON configuration files decouples the firmware logic from the settings. This allows the same program to run on different devices, loading custom parameters from their respective configuration files on boot.

## Try This! (Challenges)
1. **Device Rename CLI**: Check the UART input on boot. If the user types a new name within 3 seconds, update the `"device_name"` in the JSON file and reboot.
2. **Alert Strobe**: Sound a buzzer on GP15 if the JSON parsing fails (e.g. if the file contains syntax errors).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Config file corrupted during save | Power loss during write | In critical systems, save the new config to `config_temp.json` first, verify its integrity, then rename it to overwrite `config.json` to prevent partial-write corruption. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [105 - Pico Thermostat](../intermediate/105-pico-thermostat.md)
- [169 - Pico Smart Thermostat EEPROM](169-pico-smart-thermostat-eeprom.md)
