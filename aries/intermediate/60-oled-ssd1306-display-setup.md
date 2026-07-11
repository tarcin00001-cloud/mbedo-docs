# 60 - OLED SSD1306 Display Setup (I2C)

Initialize, configure, and display text labels on an SSD1306 OLED graphic display using I2C communication on the VEGA ARIES v3 board.

## Goal
Learn how to configure the Adafruit SSD1306 library, initialize graphic screen buffers, select text sizes, and draw characters on a 128x64 pixel screen without using loops inside C++ code.

## What You Will Build
A digital graphic readout screen displaying a project header "ARIES v3" in large text and a status subtitle "System Active" below it.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| 0.96" SSD1306 OLED Display | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| SSD1306 OLED | VCC | 3V3 | Red | OLED power supply (3.3V) |
| SSD1306 OLED | GND | GND | Black | Ground reference |
| SSD1306 OLED | SDA | SDA0 (GP17) | Blue | I2C Data line |
| SSD1306 OLED | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** The SSD1306 OLED runs on 3.3V power (do not connect VCC to 5V). SDA and SCL connect to ARIES hardware I2C0 pins GP17 and GP16.

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

void setup() {
  Wire.begin();

  // Initialize the OLED display hardware
  display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR);

  display.clearDisplay();

  // Display header text (size 2 = 12x16 pixels per character)
  display.setTextSize(2);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(10, 10);
  display.print("ARIES v3");

  // Display status subtitle text (size 1 = 6x8 pixels per character)
  display.setTextSize(1);
  display.setCursor(10, 40);
  display.print("System Active");

  display.display(); // Push buffer to screen
}

void loop() {
  delay(100);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** and **SSD1306 OLED** components onto the canvas.
2. Wire the OLED: **VCC** to **3V3**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
OLED SSD1306 Initialized at 0x3C.
```

## Expected Canvas Behavior
* The OLED display lights up and shows `ARIES v3` in large white letters and `System Active` in smaller letters underneath.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Adafruit_SSD1306 display(...)` | Configures the display size and selects the I2C interface bus. |
| `display.begin(...)` | Sends initial configurations to the SSD1306 driver to turn on internal charge pumps. |
| `display.clearDisplay()` | Clears the internal frame buffer (sets all pixels to black). |
| `display.setTextSize(2)` | Configures text rendering size factor. |
| `display.display()` | Transmits the local screen buffer to the physical OLED display controller via I2C. |

## Hardware & Safety Concept
* **SSD1306 Driver and Buffer**: The SSD1306 uses a 1KB local frame buffer in the microcontroller's SRAM (128 x 64 bits = 8192 bits = 1024 bytes). Drawing commands only modify this local buffer. A separate `display.display()` call is needed to transfer the modified pixels to the screen.
* **3.3V Logic Tolerance**: The SSD1306 display controller is a 3.3V logic device. Always power it from the 3.3V supply pin on the ARIES board to prevent over-voltage damage.

## Try This! (Challenges)
1. **Draw Divider Line**: Use `display.drawLine(0, 32, 127, 32, SSD1306_WHITE)` in setup to draw a horizontal line dividing the header and subtitle.
2. **Inverted Display**: Use `display.invertDisplay(true)` to flip the display from white-on-black to black-on-white.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| OLED screen remains completely black | Power wired incorrectly or wrong address | Ensure VCC is connected to 3.3V and the address parameter in code is `0x3C` (default). |
| Text is written in code but does not show | Missing `display()` | Ensure `display.display()` is called at the end of setup to push the frame buffer. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [58 - 16x2 I2C LCD Hello World](58-16x2-i2c-lcd-hello-world.md)
- [61 - OLED Text Aligner HUD](61-oled-text-aligner-hud.md)
