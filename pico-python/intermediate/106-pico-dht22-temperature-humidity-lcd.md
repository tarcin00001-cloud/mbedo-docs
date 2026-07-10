# 106 - Pico DHT22 Temperature Humidity LCD

Build a real-time temperature and humidity display console that reads a DHT22 sensor and prints live climate values on an I2C character LCD screen.

## Goal
Learn how to initialize a DHT22 sensor, read calibrated temperature and humidity values, handle read exceptions, and display the formatted climate data on an I2C character LCD in MicroPython.

## What You Will Build
A climate monitoring console:
- **DHT22 Sensor (GP16)**: Reads temperature (°C) and relative humidity (%).
- **I2C 16x2 LCD (GP4, GP5)**: Displays temperature on line 1 and humidity on line 2.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 | VCC (+) | 3.3V (3V3) | Red | Power line |
| DHT22 | GND (−) | GND | Black | Ground reference |
| DHT22 | DATA | GP16 | Yellow | 1-Wire data signal |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the DHT22 DATA pin to GP16 (with an optional 10 kΩ pull-up resistor to 3.3V on the DATA line for longer cable runs). Connect the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime
import dht
from machine_lcd import I2cLcd

# Initialize DHT22 sensor on GP16
sensor = dht.DHT22(Pin(16))

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("Climate Monitor")
lcd.move_to(0, 1)
lcd.putstr("Initializing...")
utime.sleep(2) # DHT warmup delay

print("DHT22 Climate Monitor active.")

while True:
    try:
        sensor.measure()
        temp_c  = sensor.temperature()
        humid   = sensor.humidity()
        
        # Update LCD Display
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("Temp: {:.1f} C".format(temp_c))
        lcd.move_to(0, 1)
        lcd.putstr("Hum:  {:.1f} %".format(humid))
        
        print("Temp: {:.1f} C | Humid: {:.1f} %".format(temp_c, humid))
    except OSError as e:
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("Sensor Error!")
        lcd.move_to(0, 1)
        lcd.putstr("Check wiring...")
        print(">> Sensor read error:", e)
        
    utime.sleep(2) # DHT22 minimum read interval (2 seconds)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22 Sensor**, and **I2C LCD** onto the canvas.
2. Connect DHT22 DATA to **GP16** and LCD to **GP4/GP5**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Observe the climate values updating on the LCD screen every 2 seconds.

## Expected Output
```
DHT22 Climate Monitor active.
Temp: 24.5 C | Humid: 58.2 %
Temp: 24.6 C | Humid: 58.0 %
```
(On screen: "Temp: 24.5 C" on line 1 and "Hum: 58.2 %" on line 2, updating every 2 seconds.)

## Expected Canvas Behavior
- The LCD component on the canvas displays live temperature and humidity values that change when the DHT22 sensor sliders are adjusted.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `sensor.measure()` | Triggers a new one-wire measurement cycle on the DHT22 sensor. |
| `sensor.temperature()` | Returns the last measured temperature in degrees Celsius. |
| `"Temp: {:.1f} C".format(temp_c)` | Formats the float value to 1 decimal place for display on the LCD. |

## Hardware & Safety Concept: DHT22 Minimum Sampling Interval
The DHT22 sensor requires at least **2 seconds** between successive measurements. The sensor's internal capacitor charges and discharges to capture accurate humidity values. Polling the sensor faster than this minimum interval will cause read failures or return stale data from the last valid reading.

## Try This! (Challenges)
1. **Comfort Index**: Add a third calculation for the Heat Index (feels-like temperature) using the temperature and humidity values, and display it on a second LCD.
2. **Trend Arrows**: Compare the current temperature reading with the previous one and display an up/down arrow character on the LCD to show the trend direction.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| "Sensor Error!" on LCD | DATA pin issue | Check that the DHT22 DATA pin is wired to GP16 and that a 10 kΩ pull-up resistor is present. |
| Values always the same | Read interval too fast | Ensure the `utime.sleep(2)` delay is at least 2 seconds between calls to `sensor.measure()`. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [13 - Pico DHT22 Serial](13-pico-dht22-serial.md)
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [52 - Pico LCD Weather Station](52-pico-lcd-weather-station.md)
