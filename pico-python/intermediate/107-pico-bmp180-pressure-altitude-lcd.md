# 107 - Pico BMP180 Pressure Altitude LCD

Build a barometric pressure and altitude display console that reads a BMP180 sensor over I2C and shows live pressure and calculated altitude on an I2C character LCD screen.

## Goal
Learn how to initialize a BMP180 barometric sensor over I2C, read calibrated pressure values, calculate altitude from pressure, and display the formatted data on an I2C character LCD in MicroPython.

## What You Will Build
A barometric monitoring console:
- **BMP180 Sensor (GP4, GP5)**: Reads barometric pressure (hPa) and temperature.
- **I2C 16x2 LCD (GP4, GP5)**: Displays pressure on line 1 and calculated altitude on line 2 (shared I2C bus).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| BMP180 Sensor | `bmp180` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| BMP180 | VCC (+) | 3.3V (3V3) | Red | Power line |
| BMP180 | GND (−) | GND | Black | Ground reference |
| BMP180 | SDA | GP4 | Orange | Shared I2C data bus |
| BMP180 | SCL | GP5 | Blue | Shared I2C clock bus |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The BMP180 and LCD share the I2C bus on GP4 (SDA) and GP5 (SCL). Each device has a unique I2C address (BMP180 = 0x77, LCD = 0x27), so they operate independently on the same two wires. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime
import math
from bmp085 import BMP180
from machine_lcd import I2cLcd

# Initialize shared I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)

# Initialize BMP180 sensor (address 0x77 by default)
bmp = BMP180(i2c)
bmp.oversample_setting = 2

# Initialize LCD (address 0x27 by default)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Standard sea-level pressure (hPa) used as reference
SEA_LEVEL_PA = 101325.0

def calc_altitude(pressure_pa):
    """Calculate altitude in metres from pressure using barometric formula."""
    return 44330.0 * (1.0 - (pressure_pa / SEA_LEVEL_PA) ** (1.0 / 5.255))

lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("Baro Console")
lcd.move_to(0, 1)
lcd.putstr("Initializing...")
utime.sleep(1)

print("BMP180 Barometric Monitor active.")

while True:
    pressure_pa = bmp.pressure     # Pressure in Pascals
    temp_c      = bmp.temperature  # Temperature in Celsius
    
    pressure_hpa = pressure_pa / 100.0
    altitude_m   = calc_altitude(pressure_pa)
    
    # Update LCD Display
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("P:{:.1f}hPa".format(pressure_hpa))
    lcd.move_to(0, 1)
    lcd.putstr("Alt:{:.0f}m T:{:.1f}C".format(altitude_m, temp_c))
    
    print("Pressure: {:.1f} hPa | Alt: {:.0f} m | Temp: {:.1f} C".format(
        pressure_hpa, altitude_m, temp_c))
    
    utime.sleep(2) # 2-second update interval
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **BMP180 Sensor**, and **I2C LCD** onto the canvas.
2. Connect BMP180 SDA/SCL to **GP4/GP5** and LCD SDA/SCL to **GP4/GP5** (shared bus). Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Observe the pressure and altitude values updating on the LCD screen every 2 seconds.

## Expected Output
```
BMP180 Barometric Monitor active.
Pressure: 1013.2 hPa | Alt: 0 m | Temp: 24.3 C
Pressure: 1013.0 hPa | Alt: 2 m | Temp: 24.3 C
```
(On screen: "P:1013.2hPa" on line 1 and "Alt:0m T:24.3C" on line 2, updating every 2 seconds.)

## Expected Canvas Behavior
- The LCD component on the canvas displays live pressure and altitude values that change when the BMP180 sensor sliders are adjusted.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `bmp.pressure` | Returns the raw measured atmospheric pressure in Pascals. |
| `calc_altitude(pressure_pa)` | Applies the international barometric formula to convert pressure to metres above sea level. |
| `pressure_pa / 100.0` | Converts Pascals to hectopascals (hPa), the standard meteorological unit. |

## Hardware & Safety Concept: Shared I2C Bus Addressing
Multiple I2C devices can share the same two-wire bus because each device responds only to its unique 7-bit **I2C address**. The master (Pico) addresses each device individually by sending its address byte at the start of every transaction. This allows sensors, displays, and EEPROMs to share a single wire pair, saving GPIO pins.

## Try This! (Challenges)
1. **Weather Trend**: Store the last 10 pressure readings and display a "RISING", "FALLING", or "STABLE" trend indicator on the LCD.
2. **Altitude Alarm**: Add a buzzer on GP15 that sounds if the calculated altitude exceeds a programmable limit (simulating a ceiling breach alarm).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| "OSError: [Errno 19]" on boot | BMP180 not found on I2C bus | Run an I2C scan to verify the sensor address. Check SDA/SCL wiring. |
| LCD shows garbled text | I2C bus conflict | Ensure both devices use unique addresses. Run `i2c.scan()` to confirm. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [14 - Pico BMP180 Serial](14-pico-bmp180-serial.md)
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [53 - Pico LCD DHT22 BMP180](53-pico-lcd-dht22-bmp180.md)
