# 162 - Pico Weather Station Datalogger

Build an outdoor meteorological tracking station that pools DHT22 climate, BMP180 barometric pressure, LDR sun level, and rain sensor datasets to stream clean CSV logs to Serial.

## Goal
Learn how to pool multiple analog and digital inputs, display status summaries on I2C screens, and stream multi-column CSV datasets to the Serial Monitor using MicroPython.

## What You Will Build
An outdoor environmental datalogger console:
- **DHT22 Sensor (GP12)**: Measures temperature and humidity.
- **BMP180 Sensor (GP4, GP5)**: Measures barometric pressure.
- **LDR Sensor (GP26)**: Measures sunlight intensity.
- **Rain Sensor (GP27)**: Measures rain levels.
- **16x2 I2C LCD (GP4, GP5)**: Displays key metrics (e.g. Temp/Hum and Rain status).
- **Serial Datalogger**: Streams CSV-formatted lines (e.g. `Temp,Hum,Pres,Sun,Rain`) every 5 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| BMP180 Barometric Sensor | `bmp180` | Yes | Yes |
| LDR Photoresistor | `ldr` | Yes | Yes |
| Rain Drop Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 | DATA | GP12 | Yellow | Climate sensor data |
| DHT22 | VCC / GND | 3.3V / GND | Red / Black | Power lines |
| BMP180 | SDA / SCL | GP4 / GP5 | Orange / Blue | Shared I2C Bus 0 |
| BMP180 | VCC / GND | 3.3V / GND | Red / Black | Power lines |
| LDR Sensor | Signal AO | GP26 | Green | Light sensor analog input |
| Rain Sensor | Signal AO | GP27 | Purple | Rain sensor analog input |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C display bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The BMP180 and I2C LCD share I2C Bus 0 (GP4/GP5) in parallel. DHT22 connects to GP12. LDR and Rain Sensors connect to GP26 and GP27 respectively. All modules must share a common ground with the Pico.

## Code
```python
from machine import Pin, ADC, I2C
import utime, dht
from bmp085 import BMP180
from machine_lcd import I2cLcd

dht_sensor = dht.DHT22(Pin(12))
ldr_sensor = ADC(26)
rain_sensor = ADC(27)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)
bmp = BMP180(i2c)
bmp.oversample_setting = 2

lcd.clear()
lcd.putstr("Meteo Station\nLogging Active")
print("Temp_C,Hum_pct,Pres_hPa,Light_idx,Rain_idx")
utime.sleep(1.5)

while True:
    try:
        dht_sensor.measure()
        temp = dht_sensor.temperature()
        humid = dht_sensor.humidity()
    except OSError:
        temp = float('nan')
        humid = float('nan')
        
    pressure = bmp.pressure / 100.0  # Pa to hPa
    light = ldr_sensor.read_u16()
    rain = rain_sensor.read_u16()
    
    if not (temp != temp or humid != humid): # Check if nan (in Python nan != nan is True)
        # 1. Update LCD Screen
        lcd.clear()
        lcd.putstr("T:{:.1f}C H:{:.0f}%\n".format(temp, humid))
        
        rain_str = "WET" if rain < 24000 else "DRY"
        lcd.putstr("Sun:{} R:{}".format(light // 655, rain_str))
        
        # 2. Stream CSV Logs to Serial
        print("{:.2f},{:.2f},{:.1f},{},{}".format(temp, humid, pressure, light, rain))
        
    utime.sleep(5.0)  # Log once every 5 seconds
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, **BMP180**, **LDR**, **Rain Sensor** (potentiometer), and **I2C LCD** onto the canvas.
2. Connect DHT22 to **GP12**, BMP180 and LCD to **GP4/GP5** in parallel, LDR to **GP26**, and Rain Sensor to **GP27**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the sensor sliders and watch the CSV logs update in the Serial terminal.

## Expected Output
Terminal:
```
Temp_C,Hum_pct,Pres_hPa,Light_idx,Rain_idx
24.50,50.20,1013.0,8000,65535
24.60,50.30,1013.0,8100,12000
```

## Expected Canvas Behavior
* Normal state (Dry): LCD reads `T:24.5C H:50%` / `Sun:12 R:DRY`.
* Rain detected (Rain sensor slider < 24000): LCD updates to `R:WET`, CSV log logs rain level.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `dht_sensor.measure()` | Triggers the DHT22 to acquire new temperature and humidity readings. |
| `bmp.pressure / 100.0` | Converts barometric pressure from Pascals to Hectopascals (hPa). |
| `print(..., ..., ...)` | Formats the sensor data into a CSV line and prints it to the Serial output. |

## Hardware & Safety Concept: CSV Formatting for Data Logging
CSV (Comma-Separated Values) is a standard format for datalogging. Streaming sensor values separated by commas with a carriage return at the end allows software on a computer (like Excel, MATLAB, or Python scripts) to capture the serial stream and save it directly to a spreadsheet for analysis.

## Try This! (Challenges)
1. **Rain Alert Beeper**: Connect a buzzer on GP14 and sound a short beep when rain is first detected.
2. **Relay Fan Actuator**: Connect a relay on GP10 and turn ON an exhaust fan if temperature exceeds 35°C.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Serial Monitor shows garbled text | Baud rate mismatch | Ensure the Serial Monitor window is configured to 9600 baud rate to match the `Serial.begin(9600)` code settings. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [107 - Pico BMP180 Pressure Altitude LCD](../intermediate/107-pico-bmp180-pressure-altitude-lcd.md)
- [140 - Pico Full Weather Console DHT22 BMP180 LDR OLED](140-pico-full-weather-console-dht22-bmp180-ldr-oled.md)
