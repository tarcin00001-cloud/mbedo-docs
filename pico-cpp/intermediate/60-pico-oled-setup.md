# 60 - Pico OLED Setup

Initialize and test a 0.96-inch I2C OLED display (SSD1306) with the Pico.

## Goal
Learn how to initialize high-resolution graphic screens, configure display buffers, and write text in multiple font sizes.

## What You Will Build
An OLED startup HUD:
- **0.96" SSD1306 OLED (GP4 SDA, GP5 SCL)**: Clears display, draws a border, and prints "Pico Online" and "SSD1306 Active".

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| SSD1306 OLED | VCC | 3.3V | Power supply |
| SSD1306 OLED | SDA | GP4 | I2C Data |
| SSD1306 OLED | SCL | GP5 | I2C Clock |
| SSD1306 OLED | GND | GND | Ground |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  // Address 0x3C is standard for 128x64 OLEDs
  if(display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.setTextSize(1);             // 1:1 pixel scale
    display.setTextColor(SSD1306_WHITE); // Draw white text
    
    display.setCursor(0, 10);
    display.println("Pico Online");
    
    display.setTextSize(2);             // Double size font
    display.setCursor(0, 30);
    display.println("SSD1306");
    
    display.display(); // Push buffer to screen hardware
  }
}

void loop() {
  delay(100);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **SSD1306 OLED** onto the canvas.
2. Connect OLED: **VCC** to **3.3V** (or **3V3**), **SDA** to **GP4**, **SCL** to **GP5**, **GND** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Observe the high-resolution text print on the OLED.

## Expected Output

Terminal:
```
Simulation active. SSD1306 OLED initialized at I2C address 0x3C.
```

## Expected Canvas Behavior
* Row 1: `Pico Online` (Small)
* Row 2: `SSD1306` (Large)

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `display.clearDisplay()` | Flushes the internal screen buffer array in Pico's RAM, setting all pixels to off (0). |
| `display.display()` | Transmits the entire 128x64 pixel buffer (1024 bytes) over the I2C bus to the SSD1306 controller chip, updating the screen pixels. |

## Hardware & Safety Concept: Display Buffer RAM
Unlike character LCDs that accept simple ASCII commands, graphic screens like the SSD1306 are addressed at the pixel level. To draw shapes, lines, or text, the microcontroller maintains a **frame buffer** in its internal RAM (128x64 pixels = 1024 bytes/1KB). All drawing commands write to this RAM buffer first. Calling `display()` sends the entire buffer to the display, preventing screen flickering.

## Try This! (Challenges)
1. **Geometric Border**: Draw a rectangular border around the screen boundary using `display.drawRect(0, 0, 128, 64, SSD1306_WHITE)` before writing text.
2. **Horizontal Line**: Draw a horizontal divider line beneath "Pico Online" using `display.drawFastHLine()`.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| OLED screen remains completely black | Missing display update call | Make sure you call `display.display()` after setting up your drawing instructions. Without it, changes stay in the Pico RAM and never reach the screen. |

## Mode Notes
This basic I2C graphics project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [58 - Pico LCD Print](58-pico-lcd-print.md)
- [61 - Pico OLED HUD](61-pico-oled-hud.md)
