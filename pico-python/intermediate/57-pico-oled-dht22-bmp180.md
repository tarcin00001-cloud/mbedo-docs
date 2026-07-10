# 57 - Pico OLED DHT22 BMP180

Build an advanced indoor weather console that displays temperature, relative humidity, and barometric pressure coordinates using a DHT22 and BMP180 on an SSD1306 OLED screen.

## Goal
Learn how to interface multiple I2C and digital sensors in parallel, format multiple data rows, and display rotating dashboards on an OLED screen in MicroPython.

## What You Will Build
An atmospheric monitoring console:
- **DHT22 Climate Sensor (GP12)**: Measures temperature and humidity.
- **BMP180 Barometer (GP4, GP5)**: Measures barometric air pressure.
- **SSD1306 OLED (GP4, GP5)**: Displays temperature and humidity on screen for 4 seconds, then rotates to show barometric pressure and estimated altitude for 4 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Climate Sensor | `dht22` | Yes | Yes |
| BMP180 Barometric Sensor | `bmp180` | Yes | Yes |
| SSD1306 OLED Display (128x64) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 | VCC | 3.3V (3V3) | Red | Power line |
| DHT22 | SDA (Data out) | GP12 | Yellow | Digital data pin |
| DHT22 | GND | GND | Black | Ground reference |
| BMP180 | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C sensor bus |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C display bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The BMP180 and SSD1306 OLED share the same I2C SDA (GP4) and SCL (GP5) lines in parallel. The DHT22 uses GP12.

## Code
```python
from machine import Pin, I2C
import utime
import dht
from bmp085 import BMP180 # Assumes driver package is present/loaded
import ssd1306

# Initialize DHT22
dht_sensor = dht.DHT22(Pin(12))

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c, addr=0x3C)
bmp_sensor = BMP180(i2c)

oled.fill(0)
oled.text("Meteo Console", 8, 20)
oled.text("Calibrating...", 8, 40)
oled.show()
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
        oled.fill(0)
        oled.fill_rect(0, 0, 128, 14, 1)
        oled.text("CLIMATE VALUES", 8, 3, 0)
        
        oled.text("Temp : " + str(round(t, 1)) + " C", 8, 22, 1)
        oled.text("Humid: " + str(round(h, 1)) + " %", 8, 38, 1)
        oled.rect(0, 0, 128, 64, 1)
        oled.show()
        utime.sleep(4)
        
        # 2. Page 2: Barometric Pressure
        oled.fill(0)
        oled.fill_rect(0, 0, 128, 14, 1)
        oled.text("BARO PRESSURE", 12, 3, 0)
        
        oled.text("Pres: " + str(round(p, 1)) + " hPa", 8, 25, 1)
        oled.text("Status: Normal", 8, 42, 1)
        oled.rect(0, 0, 128, 64, 1)
        oled.show()
        utime.sleep(4)
        
    except OSError as e:
        oled.fill(0)
        oled.text("Sensor Error", 8, 25, 1)
        oled.show()
        print("Read fail:", e)
        utime.sleep(2)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, **BMP180**, and **SSD1306 OLED** onto the canvas.
2. Connect BMP180 and OLED to **GP4/GP5** in parallel. Connect DHT22 to **GP12**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Observe the OLED page rotation.

## Expected Output
```
Console active. Displaying alternating datasets.
```
(On screen: Alternates between "Temp: X C / Humid: Y %" and "Baro Pressure: Z hPa" pages every 4 seconds.)

## Expected Canvas Behavior
- The OLED component displays alternating screens, drawing weather metrics matching the slider inputs.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `BMP180(i2c)` | Initializes the BMP180 sensor using the shared I2C bus channel. |
| `bmp_sensor.pressure / 100.0` | Converts raw Pascal pressure output into standard hectopascals (hPa). |
| `utime.sleep(4)` | Keeps each screen dashboard visible long enough for users to read it comfortably. |

## Hardware & Safety Concept: Shared I2C Bus Pull-ups
The I2C protocol uses open-drain lines that rely on **pull-up resistors** (typically 4.7 kΩ connected to 3.3V) to pull the signals high when idle. Breakout boards (like the BMP180 or SSD1306 OLED board) usually have these resistors pre-installed on PCB lines. When connecting multiple devices in parallel, avoid using too many pull-up resistors, as it can overload the bus and cause communication errors.

## Try This! (Challenges)
1. **Critical Pressure Warning**: Connect a buzzer on GP13 and sound a warning if pressure drops rapidly (storm warning).
2. **Display Sleep**: Turn OFF the OLED screen automatically if the temperature is comfortable to save battery.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Screen freezes or displays EIO errors | I2C address conflict or loose wire | Ensure both the BMP180 and OLED are wired correctly on the GP4/GP5 pins. |
| Altitudes read wildly inaccurate values | Sea-level pressure offset | Adjust the baseline sea-level pressure constant in your calculation to match local meteorological offsets. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [52 - Pico LCD Weather Station](52-pico-lcd-weather-station.md)
- [53 - Pico LCD DHT22 BMP180](53-pico-lcd-dht22-bmp180.md)
- [56 - Pico OLED DHT22](56-pico-oled-dht22.md)
