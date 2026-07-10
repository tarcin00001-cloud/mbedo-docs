# 52 - Pico LCD Weather Station

Build a real-time home weather station that displays ambient temperature and humidity data on an I2C character LCD.

## Goal
Learn how to read climate data from a DHT22 sensor, format the float values into custom strings, and update an I2C 16x2 LCD layout in MicroPython.

## What You Will Build
A desktop weather station:
- **DHT22 Climate Sensor (GP12)**: Measures ambient temperature and humidity.
- **I2C 16x2 LCD (GP4, GP5)**: Displays the active temperature on Row 1, and humidity percentage on Row 2.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Climate Sensor | `dht22` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 | VCC | 3.3V (3V3) | Red | Power line |
| DHT22 | SDA (Data out) | GP12 | Yellow | Digital data pin (requires 10k pull-up) |
| DHT22 | GND | GND | Black | Ground reference |
| I2C LCD | VCC | 5V (VBUS) | Red | Power line for backlight |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | GP4 | Orange | I2C data pin |
| I2C LCD | SCL | GP5 | Blue | I2C clock pin |

> **Wiring tip:** The DHT22 and I2C LCD share GND. The LCD uses the Pico's I2C Port 0 on GP4/GP5. The DHT22 data pin connects to GP12.

## Code
```python
from machine import Pin, I2C
import utime
import dht
from machine_lcd import I2cLcd

# Initialize DHT22 on GP12
sensor = dht.DHT22(Pin(12))

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

lcd.clear()
lcd.putstr("Weather Station")
lcd.move_to(0, 1)
lcd.putstr("Initializing...")
utime.sleep(2) # Allow DHT22 to stabilize

while True:
    try:
        # Trigger sensor read
        sensor.measure()
        temp = sensor.temperature()
        humid = sensor.humidity()
        
        # Display data on LCD
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("Temp: " + str(round(temp, 1)) + " C")
        
        lcd.move_to(0, 1)
        lcd.putstr("Humid: " + str(round(humid, 1)) + " %")
        
        print("T:", temp, "C | H:", humid, "%")
        
    except OSError as e:
        lcd.clear()
        lcd.putstr("Sensor Error")
        print("Failed to read sensor:", e)
        
    utime.sleep(3) # Update every 3 seconds (DHT22 needs at least 2s)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, and **I2C LCD** onto the canvas.
2. Connect DHT22 to **GP12**, LCD to **GP4/GP5**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the temperature and humidity controls on the DHT22 component and check the LCD.

## Expected Output
```
T: 24.5 C | H: 52.3 %
T: 25.2 C | H: 48.7 %
```
(On screen: "Temp: 24.5 C" and "Humid: 52.3 %" on respective lines.)

## Expected Canvas Behavior
- The LCD component displays the temperature and humidity values matching the positions of the DHT22 sliders.

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
2. **Pedestrian Display Switch**: Connect a button on GP13 that toggles the LCD backlight ON/OFF when pressed to save energy.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD displays "Sensor Error" | Sensor wire disconnected | Check the signal wire between the DHT22 SDA pin and GP12. |
| Reading values fluctuate | Sensor noise or wire length | Add a 10 kΩ pull-up resistor between the DHT22 data line and the 3.3V rail. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [10 - Pico LDR Light Sensor Serial](10-pico-ldr-serial.md)
- [12 - Pico NTC Thermistor Serial](12-pico-ntc-serial.md)
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
