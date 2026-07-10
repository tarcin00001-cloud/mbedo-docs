# 56 - Pico OLED DHT22

Build a real-time home comfort monitor that displays temperature and humidity data on an SSD1306 I2C OLED screen.

## Goal
Learn how to read climate data from a DHT22 sensor, format the values into text strings, and design graphical layouts on an I2C OLED display in MicroPython.

## What You Will Build
A digital climate dashboard:
- **DHT22 Climate Sensor (GP12)**: Measures ambient temperature and humidity.
- **SSD1306 OLED (GP4, GP5)**: Displays temperature and humidity on separate lines, and draws a visual status icon (like a comfort level bar).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Climate Sensor | `dht22` | Yes | Yes |
| SSD1306 OLED Display (128x64) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 | VCC | 3.3V (3V3) | Red | Power line |
| DHT22 | SDA (Data out) | GP12 | Yellow | Digital data pin |
| DHT22 | GND | GND | Black | Ground reference |
| SSD1306 OLED | VCC | 3.3V (3V3) | Red | Power line |
| SSD1306 OLED | GND | GND | Black | Ground reference |
| SSD1306 OLED | SDA | GP4 | Orange | I2C data pin |
| SSD1306 OLED | SCL | GP5 | Blue | I2C clock pin |

> **Wiring tip:** The DHT22 and OLED share GND and 3.3V rails. The OLED uses the Pico's I2C Port 0 on GP4/GP5. The DHT22 data pin connects to GP12.

## Code
```python
from machine import Pin, I2C
import utime
import dht
import ssd1306

# Initialize DHT22 on GP12
sensor = dht.DHT22(Pin(12))

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c, addr=0x3C)

oled.fill(0)
oled.text("CLIMATE MONITOR", 8, 20)
oled.text("Initializing...", 8, 40)
oled.show()
utime.sleep(2) # Stabilize DHT22

while True:
    try:
        # Trigger sensor read
        sensor.measure()
        temp = sensor.temperature()
        humid = sensor.humidity()
        
        # Display data on OLED
        oled.fill(0)
        
        # Header block
        oled.fill_rect(0, 0, 128, 14, 1)
        oled.text("CLIMATE MONITOR", 4, 3, 0)
        
        # Temp Row
        oled.text("Temp : " + str(round(temp, 1)) + " C", 8, 22, 1)
        
        # Humidity Row
        oled.text("Humid: " + str(round(humid, 1)) + " %", 8, 36, 1)
        
        # Comfort level assessment
        comfort = "COMFORTABLE"
        if temp < 18.0 or temp > 30.0:
            comfort = "UNCOMFORTABLE"
        oled.text(comfort, 8, 50, 1)
        
        oled.rect(0, 0, 128, 64, 1)
        oled.show()
        
        print("T:", temp, "C | H:", humid, "%")
        
    except OSError as e:
        oled.fill(0)
        oled.text("Sensor Error", 8, 25, 1)
        oled.show()
        print("Failed to read sensor:", e)
        
    utime.sleep(3) # Update every 3 seconds (DHT22 rate window)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, and **SSD1306 OLED** onto the canvas.
2. Connect DHT22 to **GP12**, OLED to **GP4/GP5**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the temperature and humidity controls on the DHT22 component and observe the OLED display.

## Expected Output
```
T: 24.5 C | H: 52.3 %
T: 25.2 C | H: 48.7 %
```
(On screen: A bordered dashboard displaying the current temperature, humidity, and comfort status.)

## Expected Canvas Behavior
- The OLED component displays the temperature and humidity values matching the positions of the DHT22 sliders.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `sensor.measure()` | Commands the DHT22 to take a new reading of temperature and humidity. |
| `sensor.temperature()` | Retrieves the measured temperature in degrees Celsius as a float. |
| `sensor.humidity()` | Retrieves the relative humidity as a percentage float. |
| `utime.sleep(3)` | Ensures the code does not poll the DHT22 faster than once every 2 seconds, which would cause errors. |

## Hardware & Safety Concept: DHT22 Polling Intervals
DHT22 sensors use a proprietary single-wire communication protocol to transmit climate data packets. Reading the sensor generates internal heat that can temporarily throw off its accuracy. To prevent self-heating errors, climate control firmware should limit sensor updates to once every 3 to 5 seconds.

## Try This! (Challenges)
1. **Critical High Alarm**: Connect a buzzer on GP15 and sound an alarm beep if the humidity exceeds 80% (muffiness warning).
2. **Display Sleep**: Turn OFF the OLED screen automatically if the temperature is comfortable to save battery.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| OLED displays "Sensor Error" | Sensor wire disconnected | Check the signal wire between the DHT22 SDA pin and GP12. |
| Reading values fluctuate | Sensor noise or wire length | Add a 10 kΩ pull-up resistor between the DHT22 data line and the 3.3V rail. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [10 - Pico LDR Light Sensor Serial](10-pico-ldr-serial.md)
- [12 - Pico NTC Thermistor Serial](12-pico-ntc-serial.md)
- [55 - Pico OLED Hello World](55-pico-oled-hello-world.md)
