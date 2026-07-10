# 192 - Pico Asyncio Multi-Sensor Climate Node

Build a non-blocking multi-sensor weather console that coordinates independent asynchronous read tasks for DHT22 and BMP180 sensors while managing a background alarm.

## Goal
Learn how to schedule multiple sensor polling loops at different timing intervals, manage I2C bus sharing between screens and barometers, and pulse warning alerts asynchronously.

## What You Will Build
An asynchronous meteorology terminal:
- **DHT22 Sensor (GP12)**: Polled every 3 seconds for relative humidity.
- **BMP180 Barometer (GP4, GP5)**: Polled every 2 seconds for pressure and altitude.
- **SSD1306 OLED (GP4, GP5)**: Updated at 5 Hz (every 200 ms) to display the climate telemetry.
- **Active Buzzer (GP14)**: Emits a pulsing warning siren asynchronously if temperature or pressure thresholds are breached.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| BMP180 Barometric Sensor | `bmp180` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 | DATA | GP12 | Yellow | Climate sensor data |
| DHT22 | VCC / GND | 3.3V / GND | Red / Black | Power lines |
| BMP180 Sensor | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| BMP180 Sensor | VCC / GND | 3.3V / GND | Red / Black | Power lines |
| Active Buzzer | VCC (+) | GP14 | Blue | Warning buzzer pin |
| Active Buzzer | GND | GND | Black | Ground return |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The BMP180 and SSD1306 OLED share I2C Bus 0 on GP4/GP5 in parallel. The DHT22 is wired to GP12. The active buzzer connects to GP14.

## Code
```python
import uasyncio as asyncio
from machine import Pin, I2C
import dht, ssd1306
from bmp085 import BMP180

# Hardware Setup
dht_sensor = dht.DHT22(Pin(12))
buzzer = Pin(14, Pin.OUT)
buzzer.value(0)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)
bmp = BMP180(i2c)
bmp.oversample_setting = 2

# Shared asynchronous state
temp_c = 0.0
humid_pct = 0.0
pressure_hpa = 1013.0
alarm_triggered = False

# Thresholds
TEMP_LIMIT = 32.0      # Beep alert if > 32°C
PRES_LIMIT = 1020.0    # Beep alert if > 1020 hPa

async def poll_dht22():
    """Reads DHT22 sensor every 3 seconds."""
    global temp_c, humid_pct
    print("DHT22 polling loop active.")
    while True:
        try:
            dht_sensor.measure()
            temp_c = dht_sensor.temperature()
            humid_pct = dht_sensor.humidity()
        except OSError:
            print("DHT22 read failed.")
        await asyncio.sleep(3)

async def poll_bmp180():
    """Reads BMP180 sensor every 2 seconds."""
    global pressure_hpa
    print("BMP180 polling loop active.")
    while True:
        try:
            # bmp.pressure returns pressure in Pascals
            pressure_hpa = bmp.pressure / 100.0
        except OSError:
            print("BMP180 read failed.")
        await asyncio.sleep(2)

async def monitor_safety():
    """Evaluates safety limits and pulses the buzzer asynchronously if triggered."""
    global alarm_triggered
    print("Safety monitor loop active.")
    while True:
        if temp_c > TEMP_LIMIT or pressure_hpa > PRES_LIMIT:
            alarm_triggered = True
            # Pulse the buzzer: 100ms ON, 100ms OFF
            buzzer.value(1)
            await asyncio.sleep_ms(100)
            buzzer.value(0)
            await asyncio.sleep_ms(100)
        else:
            alarm_triggered = False
            buzzer.value(0)
            await asyncio.sleep_ms(500)  # Check less frequently when safe

async def display_loop():
    """Redraws the OLED display at 5 Hz (every 200 ms)."""
    print("OLED display loop active.")
    while True:
        oled.fill(0)
        
        # Header
        oled.fill_rect(0, 0, 128, 14, 1)
        oled.text("WEATHER STATION", 4, 3, 0)
        
        oled.text("Temp   : {:.1f} C".format(temp_c), 10, 20, 1)
        oled.text("Humid  : {:.1f} %".format(humid_pct), 10, 32, 1)
        oled.text("Pres   : {:.1f} hPa".format(pressure_hpa), 10, 44, 1)
        
        # Draw status block
        if alarm_triggered:
            oled.fill_rect(0, 54, 128, 10, 1)
            oled.text("!! ALARM ACTIVE !!", 4, 55, 0)
        else:
            oled.text("System status: OK", 4, 55, 1)
            
        oled.rect(0, 0, 128, 64, 1)
        oled.show()
        
        await asyncio.sleep_ms(200)

async def main():
    # Gather and run all tasks concurrently
    await asyncio.gather(
        poll_dht22(),
        poll_bmp180(),
        monitor_safety(),
        display_loop()
    )

# Run uasyncio event loop
asyncio.run(main())
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, **BMP180**, **SSD1306 OLED**, and **Active Buzzer** onto the canvas.
2. Connect DHT22 to **GP12**, BMP180 and OLED to **GP4/GP5**, and Buzzer to **GP14**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the DHT22 temperature slider above 32°C. Verify that the buzzer pulses instantly and the OLED status block changes, while the screen and other sensors update smoothly.

## Expected Output
Terminal:
```
DHT22 polling loop active.
BMP180 polling loop active.
Safety monitor loop active.
OLED display loop active.
```

## Expected Canvas Behavior
* OLED updates consistently. The temperature, humidity, and barometric pressure values are displayed.
* Slide temperature above 32.0°C: OLED displays `!! ALARM ACTIVE !!`, buzzer beeps.
* Return temperature below 32.0°C: Buzzer silences, status reverts to `System status: OK`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `await asyncio.gather(...)` | Combines multiple asynchronous tasks into a single awaitable group, executing them concurrently. |
| `await asyncio.sleep_ms(100)` | Suspends the buzzer pulsing task for 100 milliseconds without delaying display updates or sensor reads. |

## Hardware & Safety Concept: Decoupled Multi-sensor Architectures
Industrial weather consoles monitor multiple sensors that communicate over different interfaces (e.g. 1-Wire, I2C, SPI, or Analog). Polling these sensors sequentially in a blocking loop means if one sensor crashes or hangs, the entire device stops responding. Asynchronous execution loops ensure that a crash or timeout on the DHT22 sensor does not affect the barometric monitoring or screen rendering loops.

## Try This! (Challenges)
1. **Physical Silence Button**: Connect a push button on GP13. If pressed during an active alarm, silence the buzzer for 10 seconds (snooze alarm).
2. **Dynamic Log Ticker**: Write a fifth task that logs the weather datasets to the Serial Monitor in CSV format every 10 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers but buzzer stays ON continuously | Async delay blocked | Check if you used a blocking `utime.sleep()` call somewhere in the alarm task. You must use `await asyncio.sleep_ms()`. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [140 - Pico Full Weather Console DHT22 BMP180 LDR OLED](../advanced/140-pico-full-weather-console-dht22-bmp180-ldr-oled.md)
- [172 - Pico Asyncio Multi-Tasking](172-pico-asyncio-multi-tasking.md)
