# 109 - Pico Ultrasonic Distance OLED

Build a real-time distance measurement display console that reads an HC-SR04 ultrasonic sensor and shows live distance values on an SSD1306 OLED screen.

## Goal
Learn how to trigger and read an HC-SR04 ultrasonic sensor, calculate distances using pulse timing, and display the formatted values with a graphical bar on an SSD1306 OLED in MicroPython.

## What You Will Build
A distance measurement OLED console:
- **HC-SR04 Ultrasonic Sensor (GP16 TRIG, GP17 ECHO)**: Measures distance to the nearest obstacle.
- **SSD1306 OLED (GP4, GP5)**: Displays the measured distance in centimetres and a graphical progress bar.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `hcsr04` | Yes | Yes |
| SSD1306 OLED Display (128×64) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 | VCC | 5V (VBUS) | Red | Sensor power (requires 5V) |
| HC-SR04 | GND | GND | Black | Ground reference |
| HC-SR04 | TRIG | GP16 | Orange | Trigger pulse output |
| HC-SR04 | ECHO | GP17 | Yellow | Echo pulse input |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| SSD1306 OLED | VCC / GND | 3.3V / GND | Red / Black | OLED power |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The HC-SR04 ECHO pin outputs 5V logic. On real hardware, add a 1kΩ / 2kΩ voltage divider on the ECHO line to bring it down to 3.3V for the Pico's GPIO. Connect the OLED to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime
import machine
import ssd1306

trig = Pin(16, Pin.OUT)
echo = Pin(17, Pin.IN)

# Initialize shared I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

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
        
    distance_cm = duration / 58.0
    return distance_cm

def draw_screen(dist_cm):
    oled.fill(0) # Clear screen
    
    if dist_cm < 0:
        oled.text("DISTANCE METER", 0, 0)
        oled.text("Out of Range", 16, 28)
    else:
        oled.text("DISTANCE METER", 0, 0)
        oled.text("{:.1f} cm".format(dist_cm), 20, 20)
        
        # Draw graphical bar (0 to 100 cm → 0 to 110 px)
        bar_len = min(110, int(dist_cm * 110 / 100))
        oled.rect(8, 40, 112, 12, 1)        # Border box
        oled.fill_rect(8, 40, bar_len, 12, 1) # Fill bar
        
    oled.show()

oled.fill(0)
oled.text("Distance Meter", 0, 0)
oled.text("Initializing...", 0, 20)
oled.show()
utime.sleep(1)

print("Ultrasonic OLED Monitor active.")

while True:
    dist_cm = measure_distance()
    draw_screen(dist_cm)
    
    if dist_cm < 0:
        print("Out of range")
    else:
        print("Distance: {:.1f} cm".format(dist_cm))
        
    utime.sleep_ms(300) # Update rate
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-SR04 Ultrasonic Sensor**, and **SSD1306 OLED** onto the canvas.
2. Connect HC-SR04 TRIG to **GP16** and ECHO to **GP17**. Connect OLED to **GP4/GP5**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Adjust the HC-SR04 distance slider on the canvas and observe the OLED display update.

## Expected Output
```
Ultrasonic OLED Monitor active.
Distance: 45.2 cm
Distance: 23.8 cm
```
(On OLED: "DISTANCE METER" header, distance value, and a graphical fill bar.)

## Expected Canvas Behavior
- The OLED component on the canvas displays live distance values and a proportional graphical bar that changes when the HC-SR04 sensor slider is adjusted.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `machine.time_pulse_us(echo, 1, 30000)` | Measures the echo pulse duration in microseconds with a 30 ms timeout. |
| `oled.fill_rect(8, 40, bar_len, 12, 1)` | Draws a filled rectangle proportional to the measured distance as a graphical bar. |
| `oled.show()` | Flushes the frame buffer to the OLED display hardware. |

## Hardware & Safety Concept: Double-Buffered Display Updates
OLED displays use a **frame buffer** in RAM. All drawing operations (text, rectangles, lines) modify the buffer without immediately affecting the screen. Only calling `oled.show()` copies the buffer to the display. This prevents flicker: the screen transitions cleanly from the old frame to the new frame in one operation.

## Try This! (Challenges)
1. **Radar Sweep**: Connect a servo on GP10 and sweep the HC-SR04 sensor while plotting distance bars at each angle on the OLED screen.
2. **Proximity Alert**: Add a buzzer on GP15 that increases in frequency as the measured distance decreases.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| "Out of Range" always displayed | ECHO voltage too high for Pico | Add a 1kΩ/2kΩ voltage divider on the ECHO line to reduce 5V to 3.3V for the Pico. |
| OLED display is blank | I2C address mismatch | Run `i2c.scan()` and verify the OLED is at address `0x3C`. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [55 - Pico OLED Hello World](55-pico-oled-hello-world.md)
- [58 - Pico OLED Distance HC-SR04](58-pico-oled-distance-hc-sr04.md)
- [108 - Pico Ultrasonic Distance LCD](108-pico-ultrasonic-distance-lcd.md)
