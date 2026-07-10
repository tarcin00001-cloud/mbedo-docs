# 172 - Pico Asyncio Multi-Tasking

Build an asynchronous display console that reads a DHT22 climate sensor and renders a scrolling telemetry ticker on an SSD1306 OLED screen concurrently using cooperative multitasking.

## Goal
Learn how to manage slow sensor reads (which block for several milliseconds) alongside high-frequency display redraw loops using MicroPython's `uasyncio` library to achieve fluid screen animations.

## What You Will Build
An asynchronous climate monitor:
- **DHT22 Sensor (GP12)**: Read every 2 seconds by a slow telemetry task.
- **SSD1306 OLED (GP4, GP5)**: Refreshed at 10 Hz (every 100 ms) to scroll a status message smoothly.
- **Push Button (GP14)**: Toggles display backlight state (on/off) instantly.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| SSD1306 OLED Display | `oled` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 | DATA | GP12 | Yellow | Climate sensor data |
| DHT22 | VCC / GND | 3.3V / GND | Red / Black | Power lines |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| SSD1306 OLED | VCC / GND | 3.3V / GND | Red / Black | OLED power lines |
| Button | Terminal 1 | GP14 | White | Backlight toggle |
| Button | Terminal 2 | GND | Black | Ground return |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The OLED screen and DHT22 both run on 3.3V. Connect the OLED to GP4/GP5 and the DHT22 data line to GP12. The button connects to GP14.

## Code
```python
import uasyncio as asyncio
from machine import Pin, I2C
import dht, ssd1306

dht_sensor = dht.DHT22(Pin(12))

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

btn = Pin(14, Pin.IN, Pin.PULL_UP)

# Shared global state
temp_c = 0.0
humid_pct = 0.0
sensor_error = False
display_on = True

async def read_sensor():
    """Reads the DHT22 sensor once every 2 seconds."""
    global temp_c, humid_pct, sensor_error
    print("DHT22 reader task started.")
    while True:
        try:
            dht_sensor.measure()
            temp_c = dht_sensor.temperature()
            humid_pct = dht_sensor.humidity()
            sensor_error = False
        except OSError:
            sensor_error = True
            print(">> Sensor read error!")
        # Yield for 2 seconds
        await asyncio.sleep(2)

async def update_display():
    """Redraws the OLED screen and animates a scrolling ticker at 10Hz."""
    scroll_x = 128
    ticker_text = "CLIMATE MONITOR ONLINE -- SYSTEM OK -- "
    
    print("OLED update task started.")
    while True:
        if display_on:
            oled.fill(0)
            
            # Header
            oled.rect(0, 0, 128, 14, 1)
            oled.text("TELEMETRY", 28, 3, 0)
            
            # Climate readouts
            if sensor_error:
                oled.text("SENSOR ERROR", 16, 24, 1)
            else:
                oled.text("Temp: {:.1f} C".format(temp_c), 10, 20, 1)
                oled.text("Humid: {:.1f} %".format(humid_pct), 10, 32, 1)
            
            # Scrolling Ticker (at Y=48)
            oled.hline(0, 44, 128, 1)
            oled.text(ticker_text, scroll_x, 48, 1)
            
            # Move ticker left by 2 pixels
            scroll_x -= 2
            if scroll_x < -280:  # Restart scroll
                scroll_x = 128
                
            oled.rect(0, 0, 128, 64, 1)
            oled.show()
        else:
            oled.fill(0)
            oled.show()
            
        await asyncio.sleep_ms(100)  # 10 Hz refresh rate

async def monitor_button():
    """Monitors the display toggle button."""
    global display_on
    print("Button monitor task started.")
    while True:
        if btn.value() == 0:
            display_on = not display_on
            print(">> Display toggled. Active:", display_on)
            while btn.value() == 0:
                await asyncio.sleep_ms(20)  # Debounce
        await asyncio.sleep_ms(50)

async def main():
    # Schedule concurrent coroutines
    t1 = asyncio.create_task(read_sensor())
    t2 = asyncio.create_task(update_display())
    t3 = asyncio.create_task(monitor_button())
    
    await asyncio.gather(t1, t2, t3)

# Run event loop
asyncio.run(main())
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, **SSD1306 OLED**, and **Push Button** onto the canvas.
2. Connect DHT22 to **GP12**, OLED to **GP4/GP5**, and Button to **GP14**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Observe the smooth scrolling message at the bottom of the OLED. Adjust DHT22 sliders and watch the climate readings update without interrupting the scroll animation.

## Expected Output
Terminal:
```
DHT22 reader task started.
OLED update task started.
Button monitor task started.
>> Display toggled. Active: False
>> Display toggled. Active: True
```

## Expected Canvas Behavior
* OLED display shows a solid box with static `Temp` and `Humid` readings.
* A text banner at the bottom scrolls continuously from right to left at 10 frames per second.
* Pressing the button turns the OLED display off (blank) or on instantly.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `dht_sensor.measure()` | Performs the blocking 1-Wire read transaction, which takes around 5-10ms. |
| `oled.text(..., scroll_x, 48)` | Renders text at a variable X coordinate to create a horizontal scroll effect. |

## Hardware & Safety Concept: Decoupling Display and Sensors
Many legacy embedded loops read a sensor, display the value, delay for 2 seconds, and repeat. This blocks the microcontroller and prevents any real-time screen updates or button inputs during the delay. Decoupling sensor reads (slow, low frequency) from display draws (fast, high frequency) using async tasks results in highly responsive UIs while protecting sensor life.

## Try This! (Challenges)
1. **Blink Alert**: Add an LED on GP13 and write a fourth task that blinks it slowly if temperature exceeds 30°C.
2. **Scroll Speed Controller**: Add a potentiometer on GP26 and map its value to adjust the scrolling speed (ticker refresh delay) between 50 ms and 250 ms.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Scrolling ticker stutter | Event loop blocked | Ensure that no blocking loops or delays exist in the code. Every loop must call `await asyncio.sleep()` or `await asyncio.sleep_ms()`. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [106 - Pico DHT22 Temperature Humidity LCD](../intermediate/106-pico-dht22-temperature-humidity-lcd.md)
- [140 - Pico Full Weather Console DHT22 BMP180 LDR OLED](../advanced/140-pico-full-weather-console-dht22-bmp180-ldr-oled.md)
