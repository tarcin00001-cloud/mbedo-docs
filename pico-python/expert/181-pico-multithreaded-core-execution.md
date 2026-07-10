# 181 - Pico Multithreaded Core Execution

Build a dual-core telemetry system that runs a high-speed OLED GUI on Core 0 while executing slow DHT22 climate readings on Core 1 simultaneously.

## Goal
Learn how to use MicroPython's `_thread` module to run tasks on both of the RP2040's hardware cores simultaneously, coordinate shared variables, and understand multi-core safety.

## What You Will Build
A dual-core processing station:
- **Core 0 (Main Thread)**: Reads a Potentiometer (GP26) and updates the SSD1306 OLED Display (GP4, GP5) at 20 Hz for high-speed responsiveness.
- **Core 1 (Secondary Thread)**: Executes the slow 1-Wire read operations for a DHT22 sensor (GP12) every 2 seconds.
- **SSD1306 OLED (GP4, GP5)**: Displays the active state and sensor readings.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 | DATA | GP12 | Yellow | Climate sensor data |
| DHT22 | VCC / GND | 3.3V / GND | Red / Black | Power lines |
| Potentiometer | Wiper | GP26 | Green | High-speed analog input |
| Potentiometer | Left / Right | 3.3V / GND | Red / Black | Voltage reference rails |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Core 0 will handle the OLED and potentiometer, while Core 1 handles the DHT22. Ensure I2C pins are connected to GP4/GP5.

## Code
```python
import _thread
from machine import Pin, ADC, I2C
import utime, dht, ssd1306

# Sensors & I/O
pot = ADC(26)
dht_sensor = dht.DHT22(Pin(12))

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

# Global variables shared between Core 0 and Core 1
temperature = 0.0
humidity = 0.0
sensor_active = False

def core1_thread():
    """Background task executed exclusively on Core 1."""
    global temperature, humidity, sensor_active
    print("[Core 1] Thread started. Polling DHT22...")
    while True:
        try:
            sensor_active = True
            dht_sensor.measure()
            temperature = dht_sensor.temperature()
            humidity = dht_sensor.humidity()
            sensor_active = False
        except OSError:
            sensor_active = False
            print("[Core 1] Error reading sensor!")
        
        # Poll DHT22 every 2 seconds (safe rate limit)
        utime.sleep(2)

# Start the background thread on Core 1
# start_new_thread(function, arguments_tuple)
_thread.start_new_thread(core1_thread, ())

print("[Core 0] Main loop started. Driving OLED...")

while True:
    # 1. Read potentiometer on Core 0 (immediate)
    pot_val = pot.read_u16()
    brightness_pct = pot_val * 100 // 65536
    
    # 2. Update OLED screen on Core 0
    oled.fill(0)
    
    # Header showing core allocations
    oled.fill_rect(0, 0, 128, 14, 1)
    oled.text("C0:GUI  C1:DHT22", 2, 3, 0)
    
    # Render values
    oled.text("Temp   : {:.1f} C".format(temperature), 10, 20, 1)
    oled.text("Humid  : {:.1f} %".format(humidity), 10, 32, 1)
    oled.text("Pot Val: {:3d} %".format(brightness_pct), 10, 44, 1)
    
    # Activity indicator for Core 1
    if sensor_active:
        oled.text("*C1 ACTIVE*", 10, 56, 1)
    else:
        oled.text("C1 IDLE", 10, 56, 1)
        
    oled.rect(0, 0, 128, 64, 1)
    oled.show()
    
    # Refresh screen at 20 Hz
    utime.sleep_ms(50)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, **Potentiometer**, and **SSD1306 OLED** onto the canvas.
2. Connect DHT22 to **GP12**, Potentiometer to **GP26**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the Potentiometer control. Verify that the `Pot Val` changes instantly on the OLED display at a high frame rate, while the DHT22 readings update only every 2 seconds.

## Expected Output
Terminal:
```
[Core 1] Thread started. Polling DHT22...
[Core 0] Main loop started. Driving OLED...
```

## Expected Canvas Behavior
* OLED updates at 20 frames per second. Slider changes are smooth.
* When Core 1 is reading the sensor, `*C1 ACTIVE*` flashes at the bottom.
* DHT22 values change only when its sliders are moved and the 2-second update interval ticks.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `_thread.start_new_thread(core1_thread, ())` | Spawns a new execution context on the RP2040's Core 1, running `core1_thread` in parallel. |
| `utime.sleep_ms(50)` | Controls the frame rate of the main loop on Core 0 without affecting the execution speed of Core 1. |

## Hardware & Safety Concept: Symmetric Multiprocessing (SMP)
The RP2040 features two identical ARM Cortex-M0+ processor cores. MicroPython implements Symmetric Multiprocessing, allowing developers to execute separate execution threads on each core simultaneously. This is ideal for offloading blocking tasks (like file I/O, network requests, or slow sensor transactions) to a secondary thread, ensuring the user interface remains completely smooth.

## Try This! (Challenges)
1. **Heartbeat Flasher**: Add a Green LED on GP15 and toggle it ON/OFF inside the Core 1 thread to visually indicate when Core 1 is active.
2. **Safety Cut-Off**: Add a relay on GP10, and write code on Core 0 that shuts off the relay if Core 1 fails to update the temperature within 5 seconds (heartbeat monitoring).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Microcontroller locks up or crashes on start | Memory access collision | Both threads must not access the same hardware peripheral (like the same I2C bus) at the same time. Ensure only one core calls `oled.show()`. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [116 - Pico DHT OLED HUD](../intermediate/116-pico-dht-oled-hud.md)
- [172 - Pico Asyncio Multi-Tasking](172-pico-asyncio-multi-tasking.md)
