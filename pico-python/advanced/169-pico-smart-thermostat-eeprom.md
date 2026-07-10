# 169 - Pico Smart Thermostat EEPROM

Build an adjustable smart HVAC thermostat that saves temperature setpoints in non-volatile flash memory (EEPROM) and controls heating relays.

## Goal
Learn how to persist setting values in non-volatile memory (Pico flash), parse matrix keypad inputs, read analog NTC thermistors, and toggle heating relays in MicroPython.

## What You Will Build
An adjustable digital thermostat panel:
- **NTC Thermistor (GP26)**: Measures room temperature.
- **Relay Module (GP10)**: Toggles a heater ON when the temperature falls below the target.
- **4x4 Keypad (Rows GP2, GP3, GP6, GP7; Cols GP8, GP9, GP20, GP21)**: Pressing `'A'` increments the target setpoint, and `'B'` decrements it.
- **Flash Storage**: Saves the adjusted setpoint temperature, ensuring target parameters persist across power cycles.
- **16x2 I2C LCD (GP4, GP5)**: Displays the current temperature, target setpoint, and active heater status.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| NTC Thermistor | `potentiometer` | Yes (represented by potentiometer) | Yes (10k NTC) |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 10k-ohm Resistors | `resistor` | Optional | Yes (voltage divider) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NTC Thermistor | Signal (AO) | GP26 | Yellow | Analog temperature input (with 10k pull-down) |
| NTC Thermistor | VCC / GND | 3.3V / GND | Red / Black | Power lines |
| Relay Module | IN | GP10 | Orange | Heater control switch |
| Relay Module | VCC / GND | 5V / GND | Red / Black | Power lines |
| Keypad Rows | R1–R4 | GP2, GP3, GP6, GP7 | Blue wires | Output scanning rows |
| Keypad Cols | C1–C4 | GP8, GP9, GP20, GP21 | Yellow wires | Input columns (pull-up) |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The keypad columns connect to GP8, GP9, GP20, GP21 and require internal pull-ups. The NTC thermistor connects to GP26. The Relay connects to GP10. The LCD uses GP4/GP5 (shared I2C0).

## Code
```python
from machine import Pin, ADC, I2C
import utime
from machine_lcd import I2cLcd

# Pin definitions
THERM_PIN = ADC(26)
RELAY_PIN = Pin(10, Pin.OUT)
RELAY_PIN.value(0)  # Start heater OFF

# Keypad setup
ROWS = [Pin(p, Pin.OUT) for p in (2, 3, 6, 7)]
COLS = [Pin(p, Pin.IN, Pin.PULL_UP) for p in (8, 9, 20, 21)]
KEYS = [
    ['1', '2', '3', 'A'],
    ['4', '5', '6', 'B'],
    ['7', '8', '9', 'C'],
    ['*', '0', '#', 'D']
]

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Non-volatile save/load helpers using files
def save_setpoint(val):
    try:
        with open('setpoint.txt', 'w') as f:
            f.write(str(val))
    except Exception as e:
        print("Save failed:", e)

def load_setpoint():
    try:
        with open('setpoint.txt', 'r') as f:
            val = int(f.read().strip())
            if 10 <= val <= 40:
                return val
    except Exception:
        pass
    return 22  # Default setpoint 22°C

target_setpoint = load_setpoint()

def scan_keypad():
    for r, r_pin in enumerate(ROWS):
        # Set active row LOW
        r_pin.value(0)
        for c, c_pin in enumerate(COLS):
            if c_pin.value() == 0:
                utime.sleep_ms(20)  # Debounce
                # Wait for release
                while c_pin.value() == 0:
                    utime.sleep_ms(10)
                r_pin.value(1)
                return KEYS[r][c]
        r_pin.value(1)
    return None

lcd.clear()
lcd.putstr("Smart Thermostat")
utime.sleep(1.0)
print("Non-volatile thermostat console online.")

while True:
    raw_val = THERM_PIN.read_u16()
    
    # Approximate linear temperature calculation for 10k NTC (16-bit ADC)
    temp_c = 25.0 + (32768 - raw_val) * 0.0025
    
    setpoint_changed = False
    
    # Read keypad input key
    key = scan_keypad()
    if key:
        if key == 'A':
            target_setpoint = min(40, target_setpoint + 1)
            setpoint_changed = True
            print("Setpoint increased:", target_setpoint)
        elif key == 'B':
            target_setpoint = max(10, target_setpoint - 1)
            setpoint_changed = True
            print("Setpoint decreased:", target_setpoint)
            
    # Save new setpoint to Flash if changed
    if setpoint_changed:
        save_setpoint(target_setpoint)
        
    # Update display
    lcd.clear()
    lcd.putstr("Temp: {:.1f}C\n".format(temp_c))
    lcd.putstr("Set: {}C | ".format(target_setpoint))
    
    # Thermostat relay actuation
    if temp_c < target_setpoint:
        relay.value(1)  # Turn heater ON
        lcd.putstr("Htr:ON ")
    else:
        relay.value(0)  # Turn heater OFF
        lcd.putstr("Htr:OFF")
        
    utime.sleep_ms(200)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **NTC Thermistor**, **4x4 Keypad**, **Relay**, and **I2C LCD** onto the canvas.
2. Connect Thermistor to **GP26**, Relay to **GP10**, Keypad to Row/Col pins, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Press `'A'` on the keypad to increase the target temperature, or `'B'` to decrease it, then restart the simulation and verify that the target temperature remains saved.

## Expected Output
Terminal:
```
Non-volatile thermostat console online.
Setpoint increased: 23
Setpoint increased: 24
```

## Expected Canvas Behavior
* Startup: LCD reads `Temp: 24.0C` / `Set: 22C | Htr: OFF`.
* Press `'A'` three times: Target Setpoint increases to 25°C. LCD updates to `Set: 25C | Htr: ON`. Relay clicks ON immediately.
* Restart Simulation: Thermostat boots up with the target temperature loaded from memory.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `open('setpoint.txt', 'w')` | Creates or overwrites a text file named `setpoint.txt` in the Pico's local flash filesystem. |
| `r_pin.value(0)` | Pulls a row LOW to check which column reads LOW, indicating a pressed switch. |

## Hardware & Safety Concept: Industrial Setpoint Management
Industrial climate controllers use keypads or buttons to adjust temperature setpoints. To prevent unauthorized users from tampering with setpoints, real HVAC control interfaces require entering an authorization password (PIN) before allowing changes to the temperature configuration.

## Try This! (Challenges)
1. **Critical Overheat Alarm**: Sound a buzzer on GP14 if the temperature exceeds a safety limit of 40°C.
2. **Dynamic Backlight**: Turn OFF the LCD backlight automatically if no keys are pressed for 15 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Setpoint resets to 22°C on reboot | Flash write issue | MicroPython requires calling `close()` on files to flush data. The `with` context manager handles this automatically. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [142 - Pico NTC Thermistor PID Temperature Controller](142-pico-ntc-thermistor-pid-temperature-controller.md)
- [148 - Pico Multi-Relay Scene Controller Keypad LCD](148-pico-multi-relay-scene-controller-keypad-lcd.md)
