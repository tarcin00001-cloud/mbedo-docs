# 111 - Pico DS18B20 Temperature LCD

Build a real-time temperature display console that reads a DS18B20 one-wire digital temperature sensor and shows live readings on an I2C character LCD screen.

## Goal
Learn how to initialize a DS18B20 sensor on a one-wire bus, trigger conversions, read calibrated temperature values, and display the formatted data on an I2C character LCD in MicroPython.

## What You Will Build
A precision temperature display:
- **DS18B20 Sensor (GP16)**: Reads high-accuracy digital temperature (°C).
- **I2C 16x2 LCD (GP4, GP5)**: Displays the temperature in Celsius and Fahrenheit on respective lines.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DS18B20 Temperature Sensor | `ds18b20` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 4.7 kΩ Resistor | `resistor` | Optional in MbedO | Yes (one-wire pull-up) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS18B20 | VCC (Pin 3) | 3.3V (3V3) | Red | Power line |
| DS18B20 | GND (Pin 1) | GND | Black | Ground reference |
| DS18B20 | DATA (Pin 2) | GP16 | Yellow | 1-Wire data signal |
| 4.7 kΩ Resistor | Either leg | Between GP16 and 3.3V | — | One-wire pull-up resistor |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The 4.7 kΩ pull-up resistor between the DATA line and 3.3V is **required** for one-wire communication. Without it, the DS18B20 will not respond. Connect the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime
import onewire
import ds18x20
from machine_lcd import I2cLcd

# Initialize DS18B20 sensor on GP16 one-wire bus
ow  = onewire.OneWire(Pin(16))
ds  = ds18x20.DS18X20(ow)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("Temp Monitor")
lcd.move_to(0, 1)
lcd.putstr("Scanning bus...")
utime.sleep(1)

# Scan for devices on the one-wire bus
roms = ds.scan()
print("DS18B20 devices found:", len(roms))

if not roms:
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("No sensor found!")
    lcd.move_to(0, 1)
    lcd.putstr("Check wiring...")
    raise SystemExit("No DS18B20 found on bus.")

print("Temperature Monitor active.")

while True:
    # Trigger a temperature conversion on all sensors
    ds.convert_temp()
    utime.sleep_ms(750) # Wait for 12-bit conversion (max 750 ms)
    
    # Read from the first sensor found
    temp_c = ds.read_temp(roms[0])
    temp_f = temp_c * 9.0 / 5.0 + 32.0
    
    # Update LCD Display
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("Temp: {:.2f} C".format(temp_c))
    lcd.move_to(0, 1)
    lcd.putstr("      {:.2f} F".format(temp_f))
    
    print("Temp: {:.2f} C | {:.2f} F".format(temp_c, temp_f))
    utime.sleep(1) # 1-second update interval
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DS18B20 Sensor**, and **I2C LCD** onto the canvas.
2. Connect DS18B20 DATA to **GP16** and LCD to **GP4/GP5**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Observe the temperature values updating on the LCD screen every second.

## Expected Output
```
DS18B20 devices found: 1
Temperature Monitor active.
Temp: 24.50 C | 76.10 F
```
(On screen: "Temp: 24.50 C" on line 1 and "      76.10 F" on line 2.)

## Expected Canvas Behavior
- The LCD component on the canvas displays live temperature values in both Celsius and Fahrenheit that change when the DS18B20 sensor slider is adjusted.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `ds.scan()` | Scans the one-wire bus and returns a list of ROM addresses for all connected sensors. |
| `ds.convert_temp()` | Broadcasts a temperature conversion command to all sensors on the bus simultaneously. |
| `ds.read_temp(roms[0])` | Reads the converted temperature value from the first sensor using its ROM address. |

## Hardware & Safety Concept: One-Wire Bus Addressing
The one-wire protocol allows many DS18B20 sensors to share a single data wire. Each sensor has a unique **64-bit ROM address** assigned at the factory. The master (Pico) can address each sensor individually by broadcasting its ROM address before reading. This allows precise temperature mapping across multiple points (e.g. floor, ceiling, outdoor) using only one GPIO pin.

## Try This! (Challenges)
1. **Multi-Sensor Display**: Connect a second DS18B20 on the same data line (GP16) and display both temperatures on separate LCD lines.
2. **Threshold Alarm**: Connect a buzzer on GP15 that sounds if the temperature exceeds a programmable limit.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| "No sensor found!" on LCD | Missing pull-up resistor | Ensure a 4.7 kΩ resistor is connected between the DATA line and 3.3V. |
| Reading always returns 85.0°C | Sensor not powered correctly | This is the DS18B20 power-on default. Check VCC and GND connections. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [106 - Pico DHT22 Temperature Humidity LCD](106-pico-dht22-temperature-humidity-lcd.md)
- [107 - Pico BMP180 Pressure Altitude LCD](107-pico-bmp180-pressure-altitude-lcd.md)
