# 55 - Pico OLED Hello World

Interface a high-contrast SSD1306 OLED graphic display over the I2C protocol to print text strings and draw simple shapes.

## Goal
Learn how to configure the I2C bus, load external SSD1306 pixel driver libraries, write text coordinates, and draw graphic shapes in MicroPython.

## What You Will Build
A digital graphic display console:
- **SSD1306 OLED (GP4, GP5)**: Displays a header banner, text lines, and a bordered box frame.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| SSD1306 OLED Display (128x64) | `oled` | Yes | Yes (I2C model) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| SSD1306 OLED | VCC | 3.3V (3V3) | Red | Power line |
| SSD1306 OLED | GND | GND | Black | Ground reference |
| SSD1306 OLED | SDA | GP4 | Yellow | I2C data line |
| SSD1306 OLED | SCL | GP5 | Blue | I2C clock line |

> **Wiring tip:** Connect the OLED's SDA and SCL pins to GP4 and GP5. Ensure you use the correct I2C channel (Bus 0 for GP4/GP5).

## Code
```python
from machine import Pin, I2C
import utime
import ssd1306 # Assumes driver package is present/loaded

# Initialize I2C bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)

# Screen dimensions (standard SSD1306 is 128x64 pixels)
WIDTH  = 128
HEIGHT = 64
oled = ssd1306.SSD1306_I2C(WIDTH, HEIGHT, i2c, addr=0x3C)

print("OLED Console active.")

while True:
    # Clear display buffer
    oled.fill(0)
    
    # 1. Draw a white header block (filled rectangle)
    oled.fill_rect(0, 0, 128, 14, 1)
    
    # Write text in black (0) over the white header
    # text(string, x, y, color)
    oled.text("OLED CONSOLE", 16, 3, 0)
    
    # 2. Draw body text (white pixels = 1)
    oled.text("Hello, MbedO!", 10, 24, 1)
    oled.text("MicroPython Node", 10, 38, 1)
    
    # 3. Draw a border box frame around the screen
    oled.rect(0, 0, 128, 64, 1)
    
    # Send local buffer coordinates to screen pixels
    oled.show()
    
    utime.sleep(2) # Update interval
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **SSD1306 OLED** onto the canvas.
2. Connect OLED VCC to **3.3V**, GND to **GND**, SDA to **GP4**, and SCL to **GP5**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Observe the OLED display.

## Expected Output
```
OLED Console active.
```
(On screen: A bordered box showing "OLED CONSOLE" on a white header bar, and two text lines below it.)

## Expected Canvas Behavior
- The OLED component on the canvas lights up its pixels and displays the text and border frames in real time.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `ssd1306.SSD1306_I2C(128, 64, i2c)` | Initializes the SSD1306 driver for a 128x64 pixel screen on the active I2C bus. |
| `oled.fill(0)` | Clears the local frame buffer (sets all pixels to black/off). |
| `oled.fill_rect(0, 0, 128, 14, 1)` | Draws a solid white block at coordinates (0,0) with a width of 128 and height of 14 pixels. |
| `oled.show()` | Transmits the calculated frame buffer from the Pico's memory to the screen's controller, updating the pixels. |

## Hardware & Safety Concept: OLED Frame Buffers
Graphic screens (like the SSD1306 OLED) use a **frame buffer** layout. Drawing functions (like `text()`, `rect()`, or `line()`) only modify values in the Pico's local RAM buffer. You must call `oled.show()` to transmit this buffer over the I2C bus to the screen. Modifying the buffer locally and writing it all at once prevents screen flickering.

## Try This! (Challenges)
1. **Dynamic Counter**: Add a loop variable that increments and prints its value on the OLED screen every second.
2. **Rotating Line**: Draw a diagonal line that sweeps across the screen like a clock hand.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Screen remains black | `show()` omitted in code | Ensure `oled.show()` is called at the end of your drawing routines to push the buffer to the screen. |
| Text is garbled or misaligned | Coordinates out of bounds | Remember the screen is 128 pixels wide (X: 0 to 127) and 64 pixels high (Y: 0 to 63). Keep your drawing coordinates within these bounds. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**. The `ssd1306` library is simulated internally.

## Related Projects
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [56 - Pico OLED DHT22](56-pico-oled-dht22.md)
