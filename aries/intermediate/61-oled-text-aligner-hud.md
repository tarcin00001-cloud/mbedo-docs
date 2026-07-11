# 61 - OLED Text Aligner HUD

Format and position text blocks (left-aligned, right-aligned, and centered) to build a structured Head-Up Display (HUD) on an SSD1306 OLED screen using the VEGA ARIES v3 board.

## Goal
Learn how to calculate character pixel dimensions, coordinate positions, and render aligned text blocks on a 128x64 graphic screen without using loops inside C++ code.

## What You Will Build
An OLED screen displaying three text blocks: "LEFT" aligned to the left edge, "RIGHT" aligned to the right edge, and "CENTER" positioned exactly in the middle.

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

> **Wiring tip:** Connect display VCC to 3.3V and GND to GND. Wire the I2C lines to SDA0 (GP17) and SCL0 (GP16) on the left-side analog header.

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

  display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR);
  display.clearDisplay();

  display.setTextSize(1); // Standard font size (6x8 pixels: 5x7 character + 1 blank pixel separator)
  display.setTextColor(SSD1306_WHITE);

  // 1. Left-aligned: Start drawing from the leftmost pixel column (x = 0)
  display.setCursor(0, 5);
  display.print("LEFT");

  // 2. Right-aligned: "RIGHT" has 5 characters.
  // Total pixel width = 5 chars * 6 pixels = 30 pixels.
  // Starting cursor x coordinate = Screen Width - Text Width = 128 - 30 = 98.
  display.setCursor(98, 25);
  display.print("RIGHT");

  // 3. Centered: "CENTER" has 6 characters.
  // Total pixel width = 6 chars * 6 pixels = 36 pixels.
  // Starting cursor x coordinate = (Screen Width - Text Width) / 2 = (128 - 36) / 2 = 46.
  display.setCursor(46, 45);
  display.print("CENTER");

  display.display();
}

void loop() {
  delay(100);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** and **SSD1306 OLED** components onto the canvas.
2. Connect the OLED: **VCC** to **3V3**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
OLED HUD Alignments Configured.
```

## Expected Canvas Behavior
* The OLED display lights up and shows:
  * `LEFT` printed at the top-left corner.
  * `RIGHT` printed on the middle line aligned to the right edge.
  * `CENTER` printed on the bottom line aligned precisely in the center.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `display.setTextSize(1)` | Selects font size 1, which represents 6x8 pixels grid (5x7 char + 1 spacing). |
| `display.setCursor(0, 5)` | Places left-aligned cursor at the starting horizontal pixel `0`. |
| `display.setCursor(98, 25)` | Places right-aligned cursor at pixel `98` (calculated using text length). |
| `display.setCursor(46, 45)` | Places centered cursor at pixel `46` (center calculated coordinate). |

## Hardware & Safety Concept
* **Coordinate Plane**: Graphic screens use a 2D coordinate system where the origin `(0,0)` is in the top-left corner. Horizontal position (x) increases to the right (up to 127), and vertical position (y) increases downward (up to 63).
* **Monochrome OLED Pixels**: SSD1306 OLED displays are active-matrix panels where each pixel is an organic light-emitting diode. Pixels emit their own light, meaning they require no backlight and consume current proportional to the number of lit pixels. Drawing black text on a black background consumes minimal power.

## Try This! (Challenges)
1. **Dynamic Right-Align**: Modify the code to display a count value (from `millis() / 1000`) right-aligned on the top line. Calculate the position dynamically based on digits.
2. **Grid Layout**: Draw vertical divider lines at x = 42 and x = 84 to separate the screen into three even columns.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Right-aligned text is clipped or wraps | Miscalculated pixel width | Remember each character is 6 pixels wide in size 1. A 5-character word needs 30 pixels (start at x=98). |
| Text blocks overlap vertically | Cursor Y coordinates are too close | Ensure cursor Y offsets are separated by at least 10 pixels for size 1, or 18 pixels for size 2. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [60 - OLED SSD1306 Display Setup (I2C)](60-oled-ssd1306-display-setup.md)
- [66 - Servo Position Display](66-servo-position-display.md)
