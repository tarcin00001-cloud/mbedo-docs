# 174 - Pico Hardware Timer Interrupts

Build a polling-free multi-task system that schedules sensor updates and LED heartbeats using hardware timer interrupts while keeping the main loop idle.

## Goal
Learn how to use MicroPython's `machine.Timer` class to create independent, hardware-scheduled callbacks, run background operations without polling, and avoid timing drift.

## What You Will Build
A background timer scheduler:
- **Timer 0 (Heartbeat, 1 Hz)**: Toggles a Green LED (GP15) every 1000 ms in the background.
- **Timer 1 (Sensor, 5 Hz)**: Reads an LDR (GP26) and updates a Red LED (GP16) based on light levels.
- **16x2 I2C LCD (GP4, GP5)**: Displays light intensity and interrupt fire counts, updated asynchronously.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| LDR Photoresistor | `ldr` | Yes | Yes |
| Green & Red LEDs | `led` | Yes (two LEDs) | Yes |
| 10k-ohm Resistor | `resistor` | Optional | Yes (voltage divider) |
| 330 Ω Resistors × 2 | `resistor` | Optional | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Green LED | Anode (via 330 Ω) | GP15 | Green | Heartbeat indicator |
| Green LED | Cathode | GND | Black | Ground return |
| Red LED | Anode (via 330 Ω) | GP16 | Red | Threshold alert indicator |
| Red LED | Cathode | GND | Black | Ground return |
| LDR Sensor | Signal AO | GP26 | Yellow | Ambient light input |
| LDR Sensor | VCC / GND | 3.3V / GND | Red / Black | Power lines |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** All timing is controlled by hardware. Connect LEDs to GP15 and GP16. Connect the LDR sensor output to GP26. The I2C LCD uses GP4/GP5.

## Code
```python
from machine import Pin, ADC, I2C, Timer
import utime
from machine_lcd import I2cLcd

# Hardware actuators
led_heartbeat = Pin(15, Pin.OUT)
led_alert = Pin(16, Pin.OUT)
ldr = ADC(26)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Global counters
heartbeat_ticks = 0
sensor_ticks = 0
light_level = 65535

# Instantiate hardware timers
timer_heartbeat = Timer()
timer_sensor = Timer()

def heartbeat_callback(timer):
    """Callback function triggered by Timer 0 at 1 Hz."""
    global heartbeat_ticks
    heartbeat_ticks += 1
    led_heartbeat.toggle()

def sensor_callback(timer):
    """Callback function triggered by Timer 1 at 5 Hz."""
    global sensor_ticks, light_level
    sensor_ticks += 1
    light_level = ldr.read_u16()
    
    # Simple automatic light threshold switch
    if light_level < 20000:  # Dark
        led_alert.value(1)
    else:
        led_alert.value(0)

# Initialize LCD
lcd.clear()
lcd.putstr("Timer Scheduling\nActive...")
utime.sleep(1.0)

# Start hardware timers
# period is in milliseconds
timer_heartbeat.init(freq=1, mode=Timer.PERIODIC, callback=heartbeat_callback)
timer_sensor.init(freq=5, mode=Timer.PERIODIC, callback=sensor_callback)

print("Hardware timers initialized. Main loop is now sleeping.")

while True:
    # The main thread does nothing but update the LCD display.
    # All inputs, calculations, and LED operations occur in timer interrupts.
    lcd.clear()
    lcd.putstr("LDR: {:5d}\n".format(light_level))
    lcd.putstr("H:{} S:{}".format(heartbeat_ticks, sensor_ticks))
    
    # Sleep to simulate low main thread activity
    utime.sleep_ms(500)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **LDR**, **two LEDs**, and **I2C LCD** onto the canvas.
2. Connect LDR to **GP26**, LEDs to **GP15/GP16**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the LDR slider and observe the values updating on the LCD screen, proving the timers are executing background tasks.

## Expected Output
Terminal:
```
Hardware timers initialized. Main loop is now sleeping.
```
(On LCD: `LDR: 32500` / `H:5 S:25` showing correct background counts.)

## Expected Canvas Behavior
* Green LED (GP15) toggles exactly once per second.
* Red LED (GP16) turns ON immediately when LDR light levels fall below 20000.
* LCD counter updates continually.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Timer()` | Instantiates a hardware timer object using the RP2040 hardware timer peripheral. |
| `timer.init(freq=5, ...)` | Configures the timer to fire exactly 5 times per second (periodic mode). |

## Hardware & Safety Concept: Hard vs Soft Real-Time Systems
Real-time embedded systems must guarantee that tasks complete within strict deadlines. Hardware timers use physical clock cycles to trigger interrupts, guaranteeing high timing precision regardless of software load. Relying on software loops with `utime.sleep()` makes timing dependent on execution time, which causes cumulative drift.

## Try This! (Challenges)
1. **Critical Watchdog**: Add a third timer that sounds a warning buzzer on GP14 if the sensor timer tick count stops increasing (simulating a sensor crash).
2. **Frequency Shift**: Add a button on GP13. When pressed, double the heartbeat timer frequency from 1 Hz to 2 Hz.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Pico crashes during callback execution | Too much work in ISR | Timer callback functions (ISRs) must be fast. Avoid printing, I2C writes, or floating-point calculations inside the callback. Keep them to flag toggles and pin writes. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [34 - LDR Automatic Dark Detector](../../beginner/34-pico-ldr-automatic-dark-detector-light-limit-led-on.md)
- [156 - Pico Interrupt-Driven Encoder Tachometer OLED](../advanced/156-pico-interrupt-driven-encoder-tachometer-oled.md)
