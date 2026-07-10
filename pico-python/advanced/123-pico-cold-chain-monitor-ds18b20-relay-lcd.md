# 123 - Pico Cold Chain Monitor DS18B20 Relay LCD

Build a pharmaceutical cold-chain temperature monitor that continuously reads a DS18B20 sensor, energizes a cooling relay to maintain a safe temperature window, triggers an alarm on breach, and displays the status on an I2C LCD.

## Goal
Learn how to implement a closed-loop cooling control system using a DS18B20 sensor, control a relay-driven cooling unit, detect high and low temperature breaches, trigger an alarm, and display full status on an I2C LCD in MicroPython.

## What You Will Build
A cold-chain monitoring station:
- **DS18B20 Sensor (GP16)**: Reads product temperature with high precision.
- **Relay — Cooling Unit (GP10)**: Energizes when temperature exceeds the upper safe limit.
- **Red LED (GP13)**: Flashes on breach alarm.
- **Buzzer (GP15)**: Sounds on temperature breach.
- **I2C 16x2 LCD (GP4, GP5)**: Displays current temperature, relay state, and alarm status.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DS18B20 Temperature Sensor | `ds18b20` | Yes | Yes |
| Relay Module (Cooling) | `relay` | Yes | Yes |
| Red LED | `led` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 4.7 kΩ Resistor | `resistor` | Optional in MbedO | Yes (one-wire pull-up) |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS18B20 | VCC (Pin 3) | 3.3V (3V3) | Red | Power line |
| DS18B20 | GND (Pin 1) | GND | Black | Ground reference |
| DS18B20 | DATA (Pin 2) | GP16 | Yellow | 1-Wire data signal |
| 4.7 kΩ Resistor | Either leg | GP16 to 3.3V | — | One-wire pull-up resistor |
| Relay Module | IN | GP10 | Orange | Cooling relay control signal |
| Red LED | Anode (+, via 330 Ω) | GP13 | Red | Breach alarm indicator |
| Red LED | Cathode (−) | GND | Black | Shared return path to ground |
| Active Buzzer | VCC (+) | GP15 | Red | Alarm siren output |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The 4.7 kΩ pull-up resistor between the DATA line and 3.3V is required for one-wire communication. Connect the cooling relay to GP10, the alarm LED to GP13 (through 330 Ω), the buzzer to GP15, and the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime
import onewire
import ds18x20
from machine_lcd import I2cLcd

# Initialize DS18B20 on GP16 one-wire bus
ow  = onewire.OneWire(Pin(16))
ds  = ds18x20.DS18X20(ow)

relay_cool = Pin(10, Pin.OUT)
alarm_led  = Pin(13, Pin.OUT)
buzzer     = Pin(15, Pin.OUT)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Cold-chain temperature window (°C)
TEMP_COOL_ON  = 8.0   # Activate cooling above this
TEMP_COOL_OFF = 4.0   # Deactivate cooling below this (hysteresis)
TEMP_BREACH_H = 12.0  # High breach alarm
TEMP_BREACH_L = 2.0   # Low breach alarm (too cold)

# Outputs start OFF
relay_cool.value(0)
alarm_led.value(0)
buzzer.value(0)

lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("Cold Chain Mon.")
lcd.move_to(0, 1)
lcd.putstr("Scanning bus...")
utime.sleep(1)

# Scan one-wire bus for sensors
roms = ds.scan()
if not roms:
    lcd.clear()
    lcd.putstr("No DS18B20!")
    raise SystemExit("No sensor found.")
    
print("Cold chain monitor active. Sensors:", len(roms))

# Relay state (hysteresis prevents chatter)
cooling_on = False

while True:
    ds.convert_temp()
    utime.sleep_ms(750) # Wait for 12-bit conversion
    
    temp_c = ds.read_temp(roms[0])
    
    # --- Cooling control with hysteresis ---
    if not cooling_on and temp_c > TEMP_COOL_ON:
        cooling_on = True
        relay_cool.value(1)
    elif cooling_on and temp_c < TEMP_COOL_OFF:
        cooling_on = False
        relay_cool.value(0)
        
    # --- Breach alarm ---
    breach = (temp_c > TEMP_BREACH_H) or (temp_c < TEMP_BREACH_L)
    
    if breach:
        # Pulsing alarm
        alarm_led.value(1)
        buzzer.value(1)
        utime.sleep_ms(300)
        alarm_led.value(0)
        buzzer.value(0)
    else:
        alarm_led.value(0)
        buzzer.value(0)
        
    # --- Status label ---
    if temp_c > TEMP_BREACH_H:
        status = "HIGH BREACH"
    elif temp_c < TEMP_BREACH_L:
        status = "LOW BREACH"
    elif cooling_on:
        status = "COOLING"
    else:
        status = "NOMINAL"
        
    # --- LCD Update ---
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("Temp: {:.2f} C".format(temp_c))
    lcd.move_to(0, 1)
    lcd.putstr(status + (" Cool:ON" if cooling_on else " Cool:OF"))
    
    print("Temp: {:.2f} C | Status: {}".format(temp_c, status))
    utime.sleep(1)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DS18B20**, **Relay**, **Red LED**, **Active Buzzer**, and **I2C LCD** onto the canvas.
2. Connect DS18B20 to **GP16**, Relay to **GP10**, LED to **GP13**, Buzzer to **GP15**, and LCD to **GP4/GP5**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the DS18B20 temperature slider above 8°C to activate cooling. Slide above 12°C or below 2°C to trigger the breach alarm.

## Expected Output
```
Cold chain monitor active. Sensors: 1
Temp: 6.50 C | Status: NOMINAL
Temp: 9.20 C | Status: COOLING
Temp: 13.0 C | Status: HIGH BREACH
```

## Expected Canvas Behavior
- The relay closes when temperature rises above 8°C and opens when it falls below 4°C. The red LED and buzzer pulse when a breach threshold is crossed. The LCD displays live temperature and status.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `cooling_on and temp_c < TEMP_COOL_OFF` | Hysteresis check: cooling only turns OFF when temperature drops below the lower hysteresis band (4°C), not the same threshold it turned on at (8°C). |
| `breach = (temp_c > TEMP_BREACH_H) or (temp_c < TEMP_BREACH_L)` | OR gate: alarm triggers on either a high-temperature or a low-temperature breach. |

## Hardware & Safety Concept: Hysteresis in Cooling Control
Without hysteresis, a cooling relay would rapidly switch ON and OFF as the temperature oscillates around a single threshold — causing mechanical wear (relay contact arcing) and electrical interference. Hysteresis adds a dead band between the ON and OFF threshold, preventing chattering and extending relay service life.

## Try This! (Challenges)
1. **Breach Log**: Record the timestamp and temperature of each breach event in a list and display the count of breaches on the LCD.
2. **OLED Trend Graph**: Plot the last 30 temperature readings as a scrolling line graph on an SSD1306 OLED.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Reading always 85.0°C | DS18B20 power-on default | Check VCC and GND connections. Ensure the 4.7 kΩ pull-up is present. |
| Relay chatters rapidly | Missing hysteresis | Ensure `TEMP_COOL_OFF` (4°C) is lower than `TEMP_COOL_ON` (8°C). |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [111 - Pico DS18B20 Temperature LCD](111-pico-ds18b20-temperature-lcd.md)
- [103 - Pico Temperature Alarm HVAC](103-pico-temperature-alarm-hvac.md)
- [122 - Pico Greenhouse Controller DHT22 Relay LCD](122-pico-greenhouse-controller-dht22-relay-lcd.md)
