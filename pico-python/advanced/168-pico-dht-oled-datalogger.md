# 168 - Pico DHT OLED Datalogger

Build a temperature-tracking station that logs historical extremes and displays a vertical temperature scale bar on an OLED screen.

## Goal
Learn how to track climate changes over time, calculate minimum/maximum extremes without using array memory buffers, and draw relative vertical bar graphs on SSD1306 OLED displays in MicroPython.

## What You Will Build
A temperature profile visualizer:
- **DHT22 Sensor (GP12)**: Measures ambient room temperature.
- **SSD1306 OLED (GP4, GP5)**: Displays the current temperature, records historical minimum/maximum temperature limits since startup, and draws a vertical temperature scale bar graph.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 | DATA | GP12 | Yellow | Climate sensor data |
| DHT22 | VCC / GND | 3.3V / GND | Red / Black | Power lines |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| SSD1306 OLED | VCC / GND | 3.3V / GND | Red / Black | OLED power lines |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The DHT22 data line connects to GP12. The SSD1306 OLED connects to GP4 (SDA) and GP5 (SCL) of I2C Bus 0. All modules share ground.

## Code
```python
from machine import Pin, I2C
import utime, dht, ssd1306

dht_sensor = dht.DHT22(Pin(12))

i2c  = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

min_temp = 100.0
max_temp = -100.0
first_reading = True

oled.fill(0)
oled.text("Temp Logger Ready", 4, 28)
oled.show()
utime.sleep(1.0)

print("Temperature profile logger online.")

while True:
    try:
        dht_sensor.measure()
        temp = dht_sensor.temperature()
    except OSError:
        temp = float('nan')
        
    # Check for valid reading (not nan)
    if temp == temp:
        if first_reading:
            min_temp = temp
            max_temp = temp
            first_reading = False
            
        # Update extremes
        if temp < min_temp:
            min_temp = temp
        if temp > max_temp:
            max_temp = temp
            
        # Map temperature (0-40 C) to vertical bar height (0-36 pixels)
        bar_height = int(temp * 36 / 40)
        bar_height = max(0, min(36, bar_height))
        
        oled.fill(0)
        
        # Draw header bar
        oled.fill_rect(0, 0, 128, 14, 1)
        oled.text("TEMP PROFILE LOGGER", 12, 3, 0)
        
        # Display statistics
        oled.text("Current: {:.1f} C".format(temp), 10, 20, 1)
        oled.text("Min    : {:.1f} C".format(min_temp), 10, 32, 1)
        oled.text("Max    : {:.1f} C".format(max_temp), 10, 44, 1)
        
        # Draw vertical temperature scale bar graph on the right side
        oled.rect(105, 18, 14, 40, 1)  # Frame
        if bar_height > 4:
            # Draw fill inside frame
            oled.fill_rect(107, 18 + (36 - bar_height) + 2, 10, bar_height - 4, 1)
            
        oled.rect(0, 0, 128, 64, 1)
        oled.show()
        
        print("Cur:{:.1f} Min:{:.1f} Max:{:.1f}".format(temp, min_temp, max_temp))
        
    utime.sleep(2.0)  # DHT22 rate window
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, and **SSD1306 OLED** onto the canvas.
2. Connect DHT22 to **GP12**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the temperature on the DHT22 up and down, and watch the scale bar rise and fall and the minimum/maximum bounds track the changes.

## Expected Output
Terminal:
```
Temperature profile logger online.
Cur:24.0 Min:24.0 Max:24.0
Cur:35.0 Min:24.0 Max:35.0
Cur:15.0 Min:15.0 Max:35.0
```

## Expected Canvas Behavior
* Startup: OLED reads `Current: 24.0 C`, `Min: 24.0 C`, `Max: 24.0 C`. Scale bar fills to ~60% height.
* Slide temperature to 35°C: `Current` and `Max` update to 35.0°C. Scale bar rises.
* Slide temperature to 15°C: `Current` and `Min` update to 15.0°C. Scale bar falls, while `Max` remains at 35.0°C.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `temp * 36 / 40` | Scales the temperature (0 to 40°C) to fit within the 36-pixel height of the graphical bar on the OLED. |
| `oled.rect(105, 18, 14, 40, 1)` | Renders a border box outline on the OLED to represent the boundary of the scale bar. |

## Hardware & Safety Concept: Extremum Telemetry Logs
Temperature recorders are critical in medical cold chains (such as vaccine shipping boxes). If temperature levels exceed safe limits even once, the batch is compromised. These devices record extreme values (minimum and maximum) to verify that cargo stayed within safe boundaries throughout transit.

## Try This! (Challenges)
1. **Auditory Alarm**: Connect a buzzer on GP14 and sound a beep if the temperature exceeds a safety limit of 30°C.
2. **Clear Memory Button**: Connect a button on GP15 that resets the min and max temperature values to the current temperature.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Bar graph drawing overflows the screen borders | Bar offset calculation error | Ensure that the vertical fill coordinate uses the offset height mapping `18 + (36 - bar_height) + 2` to prevent drawing outside display boundaries. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [106 - Pico DHT22 Temperature Humidity LCD](../intermediate/106-pico-dht22-temperature-humidity-lcd.md)
- [140 - Pico Full Weather Console DHT22 BMP180 LDR OLED](140-pico-full-weather-console-dht22-bmp180-ldr-oled.md)
