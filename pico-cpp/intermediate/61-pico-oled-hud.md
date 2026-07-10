# 61 - Pico OLED HUD

Build a structured Heads-Up Display (HUD) layout displaying dynamic system status labels on an OLED screen.

## Goal
Learn how to create structured dashboard layouts using graphic libraries to align text labels and dynamic indicators.

## What You Will Build
An environmental monitoring console display:
- **0.96" SSD1306 OLED**: Renders a top status bar ("SYSTEM SAFE"), a middle sensor reading section ("TEMP: 24C"), and a bottom diagnostic label.

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

  if(display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.setTextColor(SSD1306_WHITE);

    // 1. Draw top header bar
    display.fillRect(0, 0, 128, 14, SSD1306_WHITE); // White header bar
    display.setTextColor(SSD1306_BLACK);           // Black text on white
    display.setTextSize(1);
    display.setCursor(30, 3);
    display.print("SYSTEM SAFE");

    // 2. Draw middle readings
    display.setTextColor(SSD1306_WHITE);           // White text on black
    display.setTextSize(1);
    display.setCursor(10, 24);
    display.print("TEMP  : 24.5 C");
    display.setCursor(10, 36);
    display.print("HUMID : 55.2 %");

    // 3. Draw bottom divider line and status footer
    display.drawFastHLine(0, 50, 128, SSD1306_WHITE);
    display.setCursor(10, 54);
    display.print("BATTERY: 100% OK");

    display.display();
  }
}

void loop() {
  delay(100);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **SSD1306 OLED** onto the canvas.
2. Connect OLED: **VCC** to **3.3V**, **SDA** to **GP4**, **SCL** to **GP5**, **GND** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Observe the structured layout (header bar, text lines, and dividing line) on the OLED.

## Expected Output

Terminal:
```
Simulation active. OLED rendering structured HUD layout.
```

## Expected Canvas Behavior
* Top: Solid white header bar containing the text `SYSTEM SAFE` in black.
* Middle: Two lines reading `TEMP  : 24.5 C` and `HUMID : 55.2 %`.
* Bottom: A horizontal dividing line and a footer reading `BATTERY: 100% OK`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `display.fillRect(0, 0, 128, 14, SSD1306_WHITE)` | Fills a rectangular block with white pixels, serving as a background banner for header text. |
| `display.setTextColor(SSD1306_BLACK)` | Inverts the text color to write black characters over the white header bar. |

## Hardware & Safety Concept: OLED Burn-in
OLED (Organic Light Emitting Diode) displays illuminate individual pixels. Keeping the same pixels lit continuously at maximum brightness for long periods causes organic material degradation, leading to **burn-in** (permanent ghosting images on the screen). To prevent burn-in on real hardware, screensavers (moving text) or timeout routines (turning the display OFF after a period of user inactivity) should be implemented.

## Try This! (Challenges)
1. **Dynamic Voltage Indicator**: Add a potentiometer to GP26 and display its voltage reading in place of the battery status on the footer.
2. **Reverse Colors**: Invert the entire layout (black header bar on white background).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Header text is invisible | Text color matches background | Ensure you switch color to `SSD1306_BLACK` when printing text inside the `fillRect` boundary. |

## Mode Notes
This basic graphic layout project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [60 - Pico OLED Setup](60-pico-oled-setup.md)
- [74 - Pico OLED Temp Graph](../../intermediate/74-pico-dht-oled.md)
