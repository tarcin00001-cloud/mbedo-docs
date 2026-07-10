# 108 - Pico Ultrasonic Distance LCD

Build a real-time distance measurement display console that reads an HC-SR04 ultrasonic sensor and shows live distance values on an I2C character LCD screen.

## Goal
Learn how to trigger and read an HC-SR04 ultrasonic sensor, calculate distances using pulse timing, and display the formatted values on an I2C character LCD in MicroPython.

## What You Will Build
A distance measurement console:
- **HC-SR04 Ultrasonic Sensor (GP16 TRIG, GP17 ECHO)**: Measures distance to the nearest obstacle.
- **I2C 16x2 LCD (GP4, GP5)**: Displays the measured distance in centimetres and a bar graph.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `hcsr04` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 | VCC | 5V (VBUS) | Red | Sensor power (requires 5V) |
| HC-SR04 | GND | GND | Black | Ground reference |
| HC-SR04 | TRIG | GP16 | Orange | Trigger pulse output |
| HC-SR04 | ECHO | GP17 | Yellow | Echo pulse input |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The HC-SR04 ECHO pin outputs 5V logic. On real hardware, add a 1kΩ / 2kΩ voltage divider on the ECHO line to bring it down to 3.3V for the Pico's GPIO. Connect the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime
import machine
from machine_lcd import I2cLcd

trig = Pin(16, Pin.OUT)
echo = Pin(17, Pin.IN)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

def measure_distance():
    """Send a trigger pulse and measure the echo duration."""
    trig.value(0)
    utime.sleep_us(2)
    trig.value(1)
    utime.sleep_us(10)
    trig.value(0)
    
    # Measure echo pulse duration (timeout = 30ms = 30000 us)
    duration = machine.time_pulse_us(echo, 1, 30000)
    
    if duration < 0:
        return -1.0 # Timeout: no echo received
        
    # Speed of sound: 343 m/s at 20°C
    # distance_cm = duration_us / 2 / 29.1
    distance_cm = duration / 58.0
    return distance_cm

lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("Distance Meter")
lcd.move_to(0, 1)
lcd.putstr("Initializing...")
utime.sleep(1)

print("Ultrasonic Distance Monitor active.")

while True:
    dist_cm = measure_distance()
    
    if dist_cm < 0:
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("Out of Range")
        lcd.move_to(0, 1)
        lcd.putstr("Dist: > 400 cm")
        print("Out of range")
    else:
        # Calculate bar length (0 to 16 chars for 0 to 100 cm)
        bar_len = min(16, int(dist_cm * 16 / 100))
        bar_str = "=" * bar_len + " " * (16 - bar_len)
        
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("Dist: {:.1f} cm".format(dist_cm))
        lcd.move_to(0, 1)
        lcd.putstr(bar_str)
        
        print("Distance: {:.1f} cm".format(dist_cm))
        
    utime.sleep_ms(300) # Update rate
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-SR04 Ultrasonic Sensor**, and **I2C LCD** onto the canvas.
2. Connect HC-SR04 TRIG to **GP16** and ECHO to **GP17**. Connect LCD to **GP4/GP5**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Adjust the HC-SR04 distance slider on the canvas and observe the LCD display update.

## Expected Output
```
Ultrasonic Distance Monitor active.
Distance: 45.2 cm
Distance: 23.8 cm
```
(On screen: "Dist: 45.2 cm" on line 1 and a bar graph of `=====` on line 2.)

## Expected Canvas Behavior
- The LCD component on the canvas displays live distance values and a proportional bar graph that changes when the HC-SR04 sensor slider is adjusted.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `machine.time_pulse_us(echo, 1, 30000)` | Waits for the echo pin to go HIGH, then measures the duration of the HIGH pulse in microseconds. |
| `duration / 58.0` | Converts the echo duration (µs) to distance (cm) using the speed-of-sound formula. |
| `bar_str = "=" * bar_len` | Creates a text-based bar graph proportional to the measured distance. |

## Hardware & Safety Concept: Ultrasonic Timeout Protection
The HC-SR04 sensor can lock the program indefinitely if no echo is received (e.g. pointing at an open space or a soft surface that absorbs the sound). The `timeout` parameter in `machine.time_pulse_us()` provides a hard time limit that prevents the main loop from hanging. Always set a timeout when reading ultrasonic sensors.

## Try This! (Challenges)
1. **Proximity Alert**: Connect a buzzer on GP15 that sounds if the measured distance falls below 20 cm.
2. **Metric/Imperial Toggle**: Connect a button on GP14 that switches the LCD display between centimetres and inches.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| "Out of Range" always displayed | ECHO voltage too high for Pico | Add a 1kΩ/2kΩ voltage divider on the ECHO line to reduce 5V to 3.3V for the Pico. |
| Distance reading is inconsistent | Soft or angled target | Ultrasonic sensors work best on flat, hard surfaces perpendicular to the beam. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [15 - Pico HC-SR04 Distance Serial](15-pico-hcsr04-serial.md)
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [54 - Pico LCD Distance HC-SR04](54-pico-lcd-distance-hc-sr04.md)
