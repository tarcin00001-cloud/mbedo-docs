# 61 - ESP32 OLED Text Aligner HUD

Build a structured heads-up display (HUD) on the SSD1306 OLED with precisely aligned labels and data fields across multiple rows — the template for all sensor dashboard UIs.

## Goal
Learn how to position text at exact pixel coordinates, mix text sizes for hierarchy, and build a professional-looking multi-field data HUD on the 128×64 OLED canvas.

## What You Will Build
A four-field HUD showing a title bar, two sensor placeholder rows (simulated with counters), and a status bar — all updated every second with clean field-by-field partial redraws to prevent flicker.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 0.96″ SSD1306 OLED (I2C) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| SSD1306 OLED | VCC | 3V3 | Red | 3.3 V supply |
| SSD1306 OLED | GND | GND | Black | Common ground |
| SSD1306 OLED | SDA | GPIO21 | Blue | I2C data |
| SSD1306 OLED | SCL | GPIO22 | Yellow | I2C clock |

> **Wiring tip:** The same wiring as Project 60 — once you have the OLED connected for that project, no rewiring is needed for this one. Re-use the same canvas setup in MbedO.

## Code
```cpp
// OLED Text Aligner HUD — structured multi-field dashboard
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH  128
#define SCREEN_HEIGHT  64
#define OLED_ADDR      0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// Draw the static HUD frame (title bar + dividers)
void drawFrame() {
  display.clearDisplay();

  // Title bar — inverted rectangle
  display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setTextSize(1);
  display.setCursor(18, 3);
  display.print("ESP32 Data HUD");

  // Horizontal dividers
  display.drawLine(0, 14, 127, 14, SSD1306_WHITE);
  display.drawLine(0, 50, 127, 50, SSD1306_WHITE);

  // Static labels
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 18); display.print("Temp:");
  display.setCursor(0, 32); display.print("Uptime:");
  display.setCursor(0, 54); display.print("Status: OK");

  display.display();
}

// Update only the data fields (avoid full clear = no flicker)
void updateData(float temp, unsigned long uptime) {
  // Clear data areas
  display.fillRect(36, 18, 92, 8, SSD1306_BLACK);
  display.fillRect(50, 32, 78, 8, SSD1306_BLACK);

  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);

  // Temperature field
  display.setCursor(36, 18);
  display.print(temp, 1); display.print(" C");

  // Uptime field
  display.setCursor(50, 32);
  display.print(uptime); display.print("s");

  display.display();
}

void setup() {
  Serial.begin(115200);
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED not found."); while (true);
  }
  drawFrame();
  Serial.println("OLED HUD ready.");
}

void loop() {
  // Simulated sensor values
  float fakeTemp   = 22.5 + (millis() / 10000.0);
  unsigned long up = millis() / 1000;

  updateData(fakeTemp, up);

  Serial.print("Temp: "); Serial.print(fakeTemp, 1);
  Serial.print("  Uptime: "); Serial.println(up);

  delay(1000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **SSD1306 OLED** onto the canvas.
2. Connect OLED **SDA** to **GPIO21**, **SCL** to **GPIO22**.
3. Paste the code and click **Run**.
4. Observe the structured HUD with inverted title bar, two data rows, and status bar.

## Expected Output
Serial Monitor:
```
OLED HUD ready.
Temp: 22.5  Uptime: 1
Temp: 22.6  Uptime: 2
Temp: 22.8  Uptime: 5
```

OLED Display Layout:
```
[ ESP32 Data HUD  ]   ← inverted title bar
Temp:  22.5 C
Uptime: 5s
──────────────────
Status: OK
```

## Expected Canvas Behavior
* Inverted title bar (white background, black text) renders at startup and stays static.
* Temperature and uptime data fields refresh every 1 second — no flicker.
* Divider lines and status bar remain static throughout.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `display.fillRect(0, 0, 128, 14, SSD1306_WHITE)` | Draws a filled white rectangle for the inverted title bar. |
| `display.setTextColor(SSD1306_BLACK)` | Switches to black text so it is visible against the white title background. |
| `display.fillRect(36, 18, 92, 8, SSD1306_BLACK)` | Clears only the data field area before rewriting — prevents ghosting. |
| `drawFrame()` called once in setup | Static elements drawn once at startup, not every loop — saves I2C bandwidth. |
| `updateData()` called every loop | Only dynamic data areas are cleared and redrawn — much faster than full clear. |

## Hardware & Safety Concept: Partial Display Updates and Flicker Reduction
Calling `display.clearDisplay()` every update erases the entire 128×64 frame buffer, then rewrites everything — during the transfer, the screen briefly goes black, causing a visible flicker at high update rates. **Partial updates** avoid this: only the pixels that change are cleared (`fillRect` with BLACK) and redrawn, while static elements remain in the buffer unchanged. The final `display.display()` transfers the complete (mostly unchanged) buffer — the static parts transmit the same bits, but at 400 kHz I2C the entire 1 KB buffer transfers in ~20 ms. This technique is used in all professional embedded UI frameworks (LVGL, u8g2, TFT_eSPI) to achieve smooth, flicker-free displays.

## Try This! (Challenges)
1. **Live sensors**: Replace `fakeTemp` with an actual NTC reading from Project 35 and show real temperature.
2. **Signal bar**: Draw a 4-bar strength indicator in the status area using four `fillRect` calls of increasing height.
3. **Three-field HUD**: Add a third data row (e.g. humidity or ADC voltage) by adding a label at `y=40` and a data clear at `y=40`.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Data field shows ghost digits | Clearance rectangle too narrow | Widen the `fillRect` width to cover the full expected number width |
| Inverted title text not visible | Text colour not set to BLACK | Set `display.setTextColor(SSD1306_BLACK)` before printing title |
| Screen flickers on every update | Using `clearDisplay()` in loop | Move static elements to `drawFrame()` in setup; only clear data fields in loop |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [60 - ESP32 OLED SSD1306 Display Setup](60-esp32-oled-ssd1306-display-setup.md)
- [74 - ESP32 DHT22 Temperature OLED Graph](74-esp32-dht22-temperature-oled-graph.md)
- [58 - ESP32 16×2 I2C LCD Print Text](58-esp32-16x2-i2c-lcd-print-text.md)
