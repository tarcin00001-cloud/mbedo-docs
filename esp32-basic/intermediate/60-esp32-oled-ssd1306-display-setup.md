# 60 - ESP32 OLED SSD1306 Display Setup

Initialise and use a 0.96″ SSD1306 OLED display over I2C to render text, lines, and simple graphics — the foundation for all OLED-based projects.

## Goal
Learn how to set up the Adafruit SSD1306 and GFX libraries, draw text at different sizes, draw geometric primitives, and structure a display update loop for the OLED.

## What You Will Build
An SSD1306 OLED (128×64 pixels) displaying a project title in large text, a horizontal divider line, and a live millisecond counter in small text — refreshing every 500 ms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 0.96″ SSD1306 OLED (I2C) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| SSD1306 OLED | VCC | 3V3 | Red | OLED operates on 3.3 V |
| SSD1306 OLED | GND | GND | Black | Common ground |
| SSD1306 OLED | SDA | GPIO21 | Blue | I2C data |
| SSD1306 OLED | SCL | GPIO22 | Yellow | I2C clock |

> **Wiring tip:** The SSD1306 OLED runs on **3.3 V** — do not connect VCC to 5 V or the display may be damaged. The default I2C address is `0x3C` for most 128×64 modules (some smaller displays use `0x3D`). GPIO 21 and 22 are the ESP32's default I2C bus pins and include internal pull-ups when I2C is active.

## Code
```cpp
// OLED SSD1306 Display Setup and Text Rendering
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

void setup() {
  Serial.begin(115200);

  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("SSD1306 not found — check wiring and I2C address.");
    while (true);   // Halt if display not found
  }

  display.clearDisplay();

  // Draw title in large text (size 2 = 12×16 px per char)
  display.setTextSize(2);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(10, 5);
  display.print("MbedO");

  // Draw horizontal divider line
  display.drawLine(0, 24, 127, 24, SSD1306_WHITE);

  display.display();   // Push buffer to screen
  Serial.println("OLED ready.");
}

void loop() {
  // Update the millisecond counter on the lower half
  display.fillRect(0, 30, 128, 34, SSD1306_BLACK);   // Clear lower region

  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 32);
  display.print("Millis: ");
  display.print(millis());

  display.setCursor(0, 48);
  display.print("ESP32 + SSD1306");

  display.display();

  Serial.print("millis: "); Serial.println(millis());
  delay(500);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **SSD1306 OLED** onto the canvas.
2. Connect OLED **SDA** to **GPIO21**, **SCL** to **GPIO22**.
3. Paste the code and click **Run**.
4. Observe the OLED widget rendering title, divider, and updating counter.

## Expected Output
Serial Monitor:
```
OLED ready.
millis: 501
millis: 1002
millis: 1504
```

OLED Display:
```
MbedO
────────────────
Millis: 1504
ESP32 + SSD1306
```

## Expected Canvas Behavior
* Title "MbedO" appears in large text on the top half of the OLED widget.
* A horizontal line separates the title from the data area.
* The millisecond counter updates every 500 ms on the lower portion.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)` | Initialises the display, enabling the internal charge pump for 3.3 V operation. |
| `display.clearDisplay()` | Fills the 128×64 pixel frame buffer with zeros (all black). |
| `display.setTextSize(2)` | Sets character size to 2× (12 wide × 16 tall pixels per character). |
| `display.drawLine(0, 24, 127, 24, SSD1306_WHITE)` | Draws a white horizontal line from pixel (0,24) to (127,24). |
| `display.fillRect(0, 30, 128, 34, SSD1306_BLACK)` | Clears the lower region before redrawing — a partial clear avoids full-screen flicker. |
| `display.display()` | Transfers the entire frame buffer to the SSD1306 over I2C. |

## Hardware & Safety Concept: OLED vs LCD Display Technology
An OLED (Organic Light-Emitting Diode) display emits light from each pixel individually — pixels that display black consume zero power. This gives true black (not backlit grey), infinite contrast ratio, and lower average power than LCD with backlight. The SSD1306 drives 128×64 = 8,192 individual pixels. Each pixel is stored in a 1-bit frame buffer (1 KB) in the microcontroller's RAM. The library renders text, lines, rectangles, and circles into this buffer using software graphics routines. The complete buffer is then transferred to the SSD1306 over I2C in a single `display()` call — this double-buffering prevents tearing artefacts during updates.

## Try This! (Challenges)
1. **Draw a rectangle**: Use `display.drawRect(10, 30, 50, 20, SSD1306_WHITE)` to add a border around a data area.
2. **Bitmap image**: Use `display.drawBitmap()` to render a 16×16 pixel logo stored as a `PROGMEM` byte array.
3. **Sensor data HUD**: Replace the millis counter with live LDR (Project 33) or NTC (Project 35) readings.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| "SSD1306 not found" in Serial Monitor | Wrong I2C address or wiring | Try address `0x3D`; verify SDA/SCL wires are not swapped |
| Display shows random pixels at startup | Missing `clearDisplay()` in setup | Always call `display.clearDisplay()` before drawing |
| Display freezes after a few updates | Missing `display.display()` call | Ensure `display.display()` is called at the end of every update cycle |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [58 - ESP32 16×2 I2C LCD Print Text](58-esp32-16x2-i2c-lcd-print-text.md)
- [61 - ESP32 OLED Text Aligner HUD](61-esp32-oled-text-aligner-hud.md)
- [74 - ESP32 DHT22 Temperature OLED Graph](74-esp32-dht22-temperature-oled-graph.md)
