# 110 - Pico DHT22 BMP180 Weather Station OLED

Build a multi-sensor weather station console that reads a DHT22 and BMP180 sensor over shared I2C and displays temperature, humidity, and pressure on an SSD1306 OLED screen.

## Goal
Learn how to initialize multiple sensors on a shared I2C bus, read calibrated climate values from a DHT22 and BMP180, and display formatted multi-line data on an SSD1306 OLED in MicroPython.

## What You Will Build
A full weather station OLED dashboard:
- **DHT22 Sensor (GP16)**: Reads temperature (°C) and relative humidity (%).
- **BMP180 Sensor (GP4, GP5)**: Reads barometric pressure (hPa) on the shared I2C bus.
- **SSD1306 OLED (GP4, GP5)**: Displays all three climate values in a formatted dashboard layout.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| BMP180 Sensor | `bmp180` | Yes | Yes |
| SSD1306 OLED Display (128×64) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 | VCC (+) | 3.3V (3V3) | Red | Power line |
| DHT22 | GND (−) | GND | Black | Ground reference |
| DHT22 | DATA | GP16 | Yellow | 1-Wire data signal |
| BMP180 | VCC (+) | 3.3V (3V3) | Red | Power line |
| BMP180 | GND (−) | GND | Black | Ground reference |
| BMP180 | SDA | GP4 | Orange | Shared I2C data bus |
| BMP180 | SCL | GP5 | Blue | Shared I2C clock bus |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| SSD1306 OLED | VCC / GND | 3.3V / GND | Red / Black | OLED power |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The BMP180 and OLED share the I2C bus on GP4/GP5. The DHT22 uses a separate single-wire protocol on GP16. All ground lines are connected to the shared ground rail.

## Code
```python
from machine import Pin, I2C
import utime
import dht
import ssd1306
from bmp085 import BMP180

# Initialize DHT22 sensor on GP16
dht_sensor = dht.DHT22(Pin(16))

# Initialize shared I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)

# Initialize BMP180 (address 0x77) and OLED (address 0x3C)
bmp = BMP180(i2c)
bmp.oversample_setting = 2

oled = ssd1306.SSD1306_I2C(128, 64, i2c)

def draw_dashboard(temp_c, humid, pressure_hpa):
    oled.fill(0)
    
    # Title bar
    oled.text("WEATHER STATION", 0, 0)
    oled.hline(0, 10, 128, 1) # Separator line
    
    # Climate readings
    oled.text("T: {:.1f} C".format(temp_c), 0, 16)
    oled.text("H: {:.1f} %".format(humid), 0, 30)
    oled.text("P: {:.1f} hPa".format(pressure_hpa), 0, 44)
    
    oled.show()

oled.fill(0)
oled.text("Weather Station", 0, 0)
oled.text("Initializing...", 0, 20)
oled.show()
utime.sleep(2) # DHT warmup delay

print("Multi-sensor weather station active.")

while True:
    try:
        # Read DHT22 sensor
        dht_sensor.measure()
        temp_c = dht_sensor.temperature()
        humid  = dht_sensor.humidity()
        
        # Read BMP180 sensor
        pressure_pa  = bmp.pressure
        pressure_hpa = pressure_pa / 100.0
        
        # Update OLED
        draw_dashboard(temp_c, humid, pressure_hpa)
        
        print("Temp: {:.1f} C | Humid: {:.1f} % | Pressure: {:.1f} hPa".format(
            temp_c, humid, pressure_hpa))
            
    except OSError as e:
        oled.fill(0)
        oled.text("SENSOR ERROR", 0, 0)
        oled.text(str(e), 0, 20)
        oled.show()
        print(">> Sensor read error:", e)
        
    utime.sleep(2) # DHT22 minimum read interval (2 seconds)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22 Sensor**, **BMP180 Sensor**, and **SSD1306 OLED** onto the canvas.
2. Connect DHT22 DATA to **GP16**. Connect BMP180 and OLED SDA/SCL to **GP4/GP5** (shared bus). Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Observe all three climate values updating on the OLED screen every 2 seconds.

## Expected Output
```
Multi-sensor weather station active.
Temp: 24.5 C | Humid: 58.2 % | Pressure: 1013.2 hPa
Temp: 24.6 C | Humid: 58.0 % | Pressure: 1013.1 hPa
```
(On OLED: "WEATHER STATION" header with separator line, followed by T/H/P readings.)

## Expected Canvas Behavior
- The OLED component on the canvas displays a formatted dashboard with live temperature, humidity, and pressure values that change when the sensor sliders are adjusted.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `dht_sensor.measure()` | Triggers a new one-wire measurement cycle on the DHT22 sensor. |
| `bmp.pressure / 100.0` | Reads barometric pressure in Pascals and converts to hPa. |
| `oled.hline(0, 10, 128, 1)` | Draws a horizontal separator line across the full 128-pixel width of the OLED. |

## Hardware & Safety Concept: Sensor Data Fusion
Combining multiple sensor readings into a single dashboard is called **sensor fusion**. High-end weather stations average multiple temperature readings from different sensors to reduce errors, since each sensor has individual measurement tolerances. Comparing readings from multiple sensors can also detect a failed sensor.

## Try This! (Challenges)
1. **Weather Trend Alert**: Calculate rolling averages of the last 5 pressure readings and display a "STORM INCOMING" message if the pressure drops by more than 5 hPa.
2. **Heat Index Calculation**: Add a heat index ("feels like") temperature calculation using the temperature and humidity values from the DHT22.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| "SENSOR ERROR" on OLED | DHT22 wiring issue | Check that the DHT22 DATA pin is wired to GP16 and that a 10 kΩ pull-up resistor is present. |
| OLED shows partial data | I2C bus conflict between BMP180 and OLED | Run `i2c.scan()` to confirm both devices appear at their expected addresses (0x77 and 0x3C). |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [55 - Pico OLED Hello World](55-pico-oled-hello-world.md)
- [57 - Pico OLED DHT22 BMP180](57-pico-oled-dht22-bmp180.md)
- [106 - Pico DHT22 Temperature Humidity LCD](106-pico-dht22-temperature-humidity-lcd.md)
- [107 - Pico BMP180 Pressure Altitude LCD](107-pico-bmp180-pressure-altitude-lcd.md)
