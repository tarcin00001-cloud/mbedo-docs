# 65 - Pico NeoPixel Chase

Create a running light chase sequence on a WS2812B NeoPixel strip without using loops.

## Goal
Learn how to program addressable LED strip animations sequentially to comply with interpreted mode constraints.

## What You Will Build
A theater-style running light chase:
- **NeoPixel Strip (4 Pixels on GP16)**: A single Red dot runs down the strip from pixel 0 to pixel 3, repeating continuously.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| WS2812B NeoPixel Strip | `neopixel` | Yes (4 pixels) | Yes (4-pixel strip) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| NeoPixel Strip | VCC | 3.3V | Power supply |
| NeoPixel Strip | DIN | GP16 | Data line |
| NeoPixel Strip | GND | GND | Ground reference |

## Code
```cpp
#include <Adafruit_NeoPixel.h>

const int NEO_PIN = 16;
const int NUM_PIXELS = 4;

Adafruit_NeoPixel strip(NUM_PIXELS, NEO_PIN, NEO_GRB + NEO_KHZ800);

int activePixel = 0;

void setup() {
  strip.begin();
  strip.show();
}

void loop() {
  // Clear all pixels first by setting them to OFF
  strip.setPixelColor(0, strip.Color(0, 0, 0));
  strip.setPixelColor(1, strip.Color(0, 0, 0));
  strip.setPixelColor(2, strip.Color(0, 0, 0));
  strip.setPixelColor(3, strip.Color(0, 0, 0));

  // Turn ON the currently active pixel in Red
  strip.setPixelColor(activePixel, strip.Color(150, 0, 0));
  strip.show();

  // Increment active pixel index
  activePixel = activePixel + 1;

  // Wrap around index at the end of the strip
  if (activePixel >= 4) {
    activePixel = 0;
  }

  delay(200); // Speed of the chase animation
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **WS2812B NeoPixel Strip** (configured to 4 pixels) onto the canvas.
2. Connect Strip: **VCC** to **3V3**, **DIN** to **GP16**, **GND** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Observe the red light chase along the pixel strip.

## Expected Output

Terminal:
```
Simulation active. Chase sequence active on GP16.
```

## Expected Canvas Behavior
| Step Time | Pixel 0 | Pixel 1 | Pixel 2 | Pixel 3 |
| --- | --- | --- | --- | --- |
| 0–200 ms | Red | OFF | OFF | OFF |
| 200–400 ms | OFF | Red | OFF | OFF |
| 400–600 ms | OFF | OFF | Red | OFF |
| 600–800 ms | OFF | OFF | OFF | Red |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `strip.setPixelColor(activePixel, ...)` | Sets the target pixel of index `activePixel` to Red in the local buffer array. |
| `activePixel = activePixel + 1` | Shifts the active LED position forward by 1 step. |

## Hardware & Safety Concept: NeoPixel Power Consumption
Each WS2812B pixel contains three tiny LEDs (Red, Green, Blue). At maximum brightness, each color draws about 20mA (total of 60mA per pixel for White). Running a 10-pixel strip at full white draws 600mA, which exceeds the current limit of the Pico's onboard regulator. Always limit brightness in code (e.g. using `150` instead of `255`) or use external power when driving larger pixel strips.

## Try This! (Challenges)
1. **Opposite Chase**: Modify the wrap-around logic so that the red dot runs backward from pixel 3 to pixel 0.
2. **Color Shift**: Change the chase color so that it alternates between Red and Blue on each pass.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Strip stays completely OFF | Index wrap check value mismatch | Ensure the active index wrap condition matches the physical number of LEDs in the strip (e.g. wrap at `4` for a 4-pixel strip). |

## Mode Notes
This basic animation project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [64 - Pico NeoPixel Fade](64-pico-neopixel-fade.md)
- [70 - Pico NeoPixel Mixer](../../intermediate/70-pico-neopixel-mixer.md)
