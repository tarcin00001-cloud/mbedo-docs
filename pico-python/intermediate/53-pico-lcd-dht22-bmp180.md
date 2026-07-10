# 53 - Pico LCD DHT22 BMP180

Build an advanced indoor weather console that displays temperature, relative humidity, and barometric pressure coordinates using a DHT22 and BMP180 on an I2C LCD.

## Goal
Learn how to interface multiple I2C and digital sensors in parallel, format multiple data rows, and display rotating dashboards on a character LCD in MicroPython.

## What You Will Build
An atmospheric monitoring console:
- **DHT22 Climate Sensor (GP12)**: Measures temperature and humidity.
- **BMP180 Barometer (GP4, GP5)**: Measures barometric air pressure.
- **I2C 16x2 LCD (GP4, GP5)**: Displays temperature and humidity on screen for 4 seconds, then rotates to show barometric pressure and estimated altitude for 4 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Climate Sensor | `dht22` | Yes | Yes |
| BMP180 Barometric Sensor | `bmp180` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 | VCC | 3.3V (3V3) | Red | Power line |
| DHT22 | SDA (Data out) | GP12 | Yellow | Digital data pin |
| DHT22 | GND | GND | Black | Ground reference |
| BMP180 | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C sensor bus |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C display bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The BMP180 and I2C LCD share the same I2C SDA (GP4) and SCL (GP5) lines in parallel. The DHT22 uses GP12.

## Code
```python
from machine import Pin, I2C
import utime
import dht
from bmp085 import BMP180 # Assumes driver package is present/loaded
from machine_lcd import I2cLcd

# Initialize DHT22
dht_sensor = dht.DHT22(Pin(12))

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)
bmp_sensor = BMP180(i2c)

lcd.clear()
lcd.putstr("Meteo Console")
lcd.move_to(0, 1)
lcd.putstr("Calibrating...")
utime.sleep(2) # Stabilize sensors

print("Console active. Displaying alternating datasets.")

while True:
    try:
        # Read sensors
        dht_sensor.measure()
        t = dht_sensor.temperature()
        h = dht_sensor.humidity()
        p = bmp_sensor.pressure / 100.0 # Convert Pascal to hPa
        
        # 1. Page 1: Temperature and Humidity
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("Temp : " + str(round(t, 1)) + " C")
        lcd.move_to(0, 1)
        lcd.putstr("Humid: " + str(round(h, 1)) + " %")
        utime.sleep(4)
        
        # 2. Page 2: Barometric Pressure
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("Baro Pressure:")
        lcd.move_to(0, 1)
        lcd.putstr(str(round(p, 1)) + " hPa")
        utime.sleep(4)
        
    except OSError as e:
        lcd.clear()
        lcd.putstr("Sensor Read Error")
        print("Read fail:", e)
        utime.sleep(2)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, **BMP180**, and **I2C LCD** onto the canvas.
2. Connect BMP180 and LCD to **GP4/GP5** in parallel. Connect DHT22 to **GP12**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Observe the LCD page rotation.

## Expected Output
```
Console active. Displaying alternating datasets.
```
(On screen: Alternates between "Temp: X C / Humid: Y %" and "Baro Pressure: Z hPa" pages every 4 seconds.)

## Expected Canvas Behavior
- The LCD component displays alternating screens, drawing weather metrics matching the slider inputs.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `BMP180(i2c)` | Initializes the BMP180 sensor using the shared I2C bus channel. |
| `bmp_sensor.pressure / 100.0` | Converts raw Pascal pressure output into standard hectopascals (hPa). |
| `utime.sleep(4)` | Keeps each screen dashboard visible long enough for users to read it comfortably. |

## Hardware & Safety Concept: Shared I2C Bus Pull-ups
The I2C protocol uses open-drain lines that rely on **pull-up resistors** (typically 4.7 kΩ connected to 3.3V) to pull the signals high when idle. Breakout boards (like the BMP180 or I2C LCD board) usually have these resistors pre-installed on PCB lines. When connecting multiple devices in parallel, avoid using too many pull-up resistors, as it can overload the bus and cause communication errors.

## Try This! (Challenges)
1. **Critical Pressure Warning**: Connect a buzzer on GP13 and sound a warning if pressure drops rapidly (storm warning).
2. **Pedestrian Display Switch**: Connect a button on GP11 to manually cycle through pages instead of rotating them automatically.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Screen freezes or displays EIO errors | I2C address conflict or loose wire | Ensure both the BMP180 and I2C LCD are wired correctly on the GP4/GP5 pins. |
| Altitudes read wildly inaccurate values | Sea-level pressure offset | Adjust the baseline sea-level pressure constant in your calculation to match local meteorological offsets. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [52 - Pico LCD Weather Station](52-pico-lcd-weather-station.md)
- [56 - Pico OLED DHT22](56-pico-oled-dht22.md)
- [57 - Pico OLED DHT22 BMP180](57-pico-oled-dht22-bmp180.md)
