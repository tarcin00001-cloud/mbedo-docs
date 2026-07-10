# 51 - Pico LCD Hello World

Interface a 16x2 character LCD using the I2C protocol to display custom text strings.

## Goal
Learn how to configure the I2C bus in MicroPython, load external character LCD driver libraries, initialize the screen, and print formatted text rows.

## What You Will Build
A digital text display terminal:
- **16x2 I2C LCD (GP4, GP5)**: Displays "Hello, MbedO!" on Row 1, and an active runtime timer on Row 2.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes (with PCF8574 I2C adapter board) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| I2C LCD Module | VCC | 5V (VBUS) or 3.3V | Red | Power line (typically 5V for bright backlight) |
| I2C LCD Module | GND | GND | Black | Ground reference |
| I2C LCD Module | SDA | GP4 | Yellow | I2C data line |
| I2C LCD Module | SCL | GP5 | Blue | I2C clock line |

> **Wiring tip:** Connect the LCD's SDA and SCL pins to GP4 and GP5. In the Raspberry Pi Pico, GP4 and GP5 correspond to I2C Port 0. Ensure the I2C power lines are connected properly.

## Code
```python
from machine import Pin, I2C
import utime
from machine_lcd import I2cLcd # Assumes driver package is present/loaded

# Initialize I2C bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)

# Typical PCF8574 I2C address is 0x27 (or 0x3F)
I2C_ADDR = 0x27
lcd = I2cLcd(i2c, I2C_ADDR, 2, 16)

print("LCD Hello World console active.")

lcd.clear()
lcd.putstr("Hello, MbedO!")

start_time = utime.ticks_ms()

while True:
    # Calculate elapsed runtime in seconds
    elapsed = utime.ticks_diff(utime.ticks_ms(), start_time) // 1000
    
    # Position cursor at Column 0, Row 1 (second line)
    lcd.move_to(0, 1)
    lcd.putstr("Runtime: " + str(elapsed) + "s  ")
    
    utime.sleep(1) # Update once per second
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **I2C LCD** onto the canvas.
2. Connect LCD VCC to **5V**, GND to **GND**, SDA to **GP4**, and SCL to **GP5**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Observe the LCD display.

## Expected Output
```
LCD Hello World console active.
```
(On screen: "Hello, MbedO!" on line 1, and "Runtime: Xs" on line 2, updating every second.)

## Expected Canvas Behavior
- The LCD component on the canvas lights up its backlight and renders the text characters in real time.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `I2C(0, sda=Pin(4), scl=Pin(5))` | Initializes I2C channel 0 on GP4 (SDA) and GP5 (SCL). |
| `I2cLcd(i2c, 0x27, 2, 16)` | Instantiates the LCD controller driver object for a 2-row, 16-column display. |
| `lcd.putstr("Hello, MbedO!")` | Prints the text string at the current cursor position. |
| `lcd.move_to(0, 1)` | Moves the drawing cursor to the start (Column 0) of the second line (Row 1). |

## Hardware & Safety Concept: I2C Address Scans
The I2C protocol allows up to 127 different devices to share the same two wire lines (SDA/SCL). Each device must have a unique **hexadecimal address** (like `0x27` for standard LCD boards). If your display does not print text on real hardware, you can run a simple I2C scanner script (`i2c.scan()`) to discover the correct address of your connected LCD module.

## Try This! (Challenges)
1. **Flashing Backlight**: Use `lcd.backlight_off()` and `lcd.backlight_on()` to flash the screen's backlight every 2 seconds.
2. **Text Scroller**: Write a function that scrolls a long text string across the first row from right to left.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Display is active but shows blocks | Contrast screw not adjusted | On real hardware, rotate the small blue potentiometer screw on the back of the I2C adapter board to adjust character contrast. |
| Code errors: `OSError: [Errno 5] EIO` | Incorrect I2C address used | Run `print([hex(x) for x in i2c.scan()])` to find the correct address of your screen and update `I2C_ADDR`. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**. The `machine_lcd` library is simulated internally to render I2C displays.

## Related Projects
- [52 - Pico LCD Weather Station](52-pico-lcd-weather-station.md)
- [55 - Pico OLED Hello World](55-pico-oled-hello-world.md)
