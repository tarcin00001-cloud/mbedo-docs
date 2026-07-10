# 136 - Pico DS18B20 Multi-Probe Temperature Network LCD

Build a multi-point temperature monitoring network that reads up to three DS18B20 sensors on a shared one-wire bus, labels each sensor by location, detects the hottest probe, and displays all readings simultaneously on an I2C LCD.

## Goal
Learn how to scan and address multiple DS18B20 sensors on a single one-wire data line, read each sensor individually by ROM address, detect the maximum temperature across all probes, and display a multi-probe dashboard on an I2C character LCD in MicroPython.

## What You Will Build
A multi-probe temperature network:
- **DS18B20 × 3 (GP16, shared bus)**: Three sensors measuring different locations (e.g. inlet, outlet, ambient).
- **I2C 16x2 LCD (GP4, GP5)**: Scrolls through all probe readings and highlights the hottest.
- **LED (GP13)**: Lights when any probe exceeds an over-temperature threshold.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DS18B20 × 3 | `ds18b20` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 4.7 kΩ Resistor | `resistor` | Optional in MbedO | Yes (single pull-up for whole bus) |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS18B20 #1 | VCC (Pin 3) | 3.3V (3V3) | Red | Shared power line |
| DS18B20 #1 | GND (Pin 1) | GND | Black | Shared ground reference |
| DS18B20 #1 | DATA (Pin 2) | GP16 | Yellow | Shared 1-Wire data line |
| DS18B20 #2 | VCC / GND | 3.3V / GND | Red / Black | Parallel power and ground |
| DS18B20 #2 | DATA (Pin 2) | GP16 | Yellow | Same shared data line |
| DS18B20 #3 | VCC / GND | 3.3V / GND | Red / Black | Parallel power and ground |
| DS18B20 #3 | DATA (Pin 2) | GP16 | Yellow | Same shared data line |
| 4.7 kΩ Resistor | Either leg | GP16 to 3.3V | — | One pull-up for entire bus |
| LED | Anode (+, via 330 Ω) | GP13 | Red | Over-temp warning indicator |
| LED | Cathode (−) | GND | Black | Ground return |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** All three DS18B20 DATA pins connect to the same GP16 wire. Only ONE 4.7 kΩ pull-up resistor is needed for the entire bus — placed between GP16 and 3.3V. Each sensor has a unique ROM address and responds to individual read commands.

## Code
```python
from machine import Pin, I2C
import utime
import onewire
import ds18x20
from machine_lcd import I2cLcd

ow  = onewire.OneWire(Pin(16))
ds  = ds18x20.DS18X20(ow)

alert_led = Pin(13, Pin.OUT)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

OVER_TEMP_C = 35.0  # Alert threshold

lcd.clear(); lcd.putstr("Probe Network")
lcd.move_to(0, 1); lcd.putstr("Scanning bus...")
utime.sleep(1)

# Scan bus
roms = ds.scan()
print("DS18B20 probes found:", len(roms))

if not roms:
    lcd.clear(); lcd.putstr("No sensors!")
    raise SystemExit("No DS18B20 found.")

# Assign location labels (extend as needed)
probe_labels = ["INLET  ", "OUTLET ", "AMBIENT"]
labels = probe_labels[:len(roms)]

lcd.clear()
lcd.putstr("{} probe(s) found".format(len(roms)))
utime.sleep(1)

print("Multi-probe network active.")

while True:
    # Trigger conversion on ALL probes simultaneously
    ds.convert_temp()
    utime.sleep_ms(750)

    readings = []
    for i, rom in enumerate(roms):
        temp = ds.read_temp(rom)
        readings.append(temp)
        print("{}: {:.2f} C".format(labels[i].strip(), temp))

    max_temp  = max(readings)
    max_probe = labels[readings.index(max_temp)].strip()
    over_temp = max_temp > OVER_TEMP_C
    alert_led.value(1 if over_temp else 0)

    # Scroll display: show two probes at a time, alternating
    # Page 1: probe 1 and 2
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("{}: {:.2f} C".format(labels[0], readings[0]) if len(readings) > 0 else "---")
    lcd.move_to(0, 1)
    lcd.putstr("{}: {:.2f} C".format(labels[1], readings[1]) if len(readings) > 1 else "---")
    utime.sleep(2)

    # Page 2: probe 3 and max
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("{}: {:.2f} C".format(labels[2], readings[2]) if len(readings) > 2 else "---")
    lcd.move_to(0, 1)
    if over_temp:
        lcd.putstr("OVER TEMP: {}".format(max_probe))
    else:
        lcd.putstr("MAX: {} {:.1f}C".format(max_probe[:3], max_temp))
    utime.sleep(2)
```

## What to Click in MbedO
1. Drag **Pico**, **three DS18B20 sensors**, **LED**, and **I2C LCD** onto the canvas.
2. Connect all three DS18B20 DATA pins to **GP16** (shared), power and GND to 3.3V/GND. Connect LED to **GP13** and LCD to **GP4/GP5**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Adjust each DS18B20 slider independently. Observe alternating LCD pages and the LED activating above 35°C on any probe.

## Expected Output
```
DS18B20 probes found: 3
Multi-probe network active.
INLET: 24.50 C
OUTLET: 28.75 C
AMBIENT: 22.00 C
```

## Expected Canvas Behavior
- The LCD alternates between two pages every 2 seconds, showing different probe readings. Sliding any DS18B20 above 35°C lights the alert LED and shows "OVER TEMP:" on page 2.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `ds.scan()` | Returns a list of 64-bit ROM addresses for all sensors on the bus. |
| `ds.convert_temp()` (broadcast) | Sends a single Skip ROM + Convert T command to all probes simultaneously, saving time. |
| `readings.index(max_temp)` | Finds which probe has the highest temperature for the MAX label display. |

## Hardware & Safety Concept: One-Wire ROM Addressing
Each DS18B20 contains a laser-etched 64-bit ROM address unique worldwide. When multiple sensors share a bus, the master uses **ROM-match commands** to select a specific sensor before reading. MicroPython's `ds18x20` library abstracts this: calling `read_temp(rom)` internally handles the ROM-match protocol.

## Try This! (Challenges)
1. **Temperature Differential**: Calculate and display the difference between INLET and OUTLET probes as a heat-exchanger efficiency metric.
2. **Rolling Average**: Track a 5-sample rolling average per probe and display the smoothed value to filter sensor noise.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Only 1 sensor detected | Missing or weak pull-up | Ensure only ONE 4.7 kΩ pull-up is present on the bus. Too many pull-ups can cause bus issues. |
| Sensor reads 85.0°C | Power-on default value | Check VCC connections. 85°C is the DS18B20 power-on register default. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [111 - Pico DS18B20 Temperature LCD](../intermediate/111-pico-ds18b20-temperature-lcd.md)
- [123 - Pico Cold Chain Monitor DS18B20 Relay LCD](123-pico-cold-chain-monitor-ds18b20-relay-lcd.md)
