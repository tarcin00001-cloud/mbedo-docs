# 64 - Pico NeoPixel Fade

Fade the brightness of a single addressable NeoPixel (WS2812B) LED.

## Goal
Learn how to initialize addressable LED protocols (WS2812B) and adjust color brightness values.

## What You Will Build
A breathing color cycle:
- **NeoPixel (WS2812B on GP16)**: Cycles Red color brightness up and down continuously.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| WS2812B NeoPixel | `neopixel` | Yes | Yes (single breakout board) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| NeoPixel | VCC | 3.3V (or 5V) | Power supply |
| NeoPixel | DIN (Data In) | GP16 | Single-wire data line |
| NeoPixel | GND | GND | Ground return |

## Code
```cpp
#include <Adafruit_NeoPixel.h>

const int NEO_PIN = 16;
const int NUM_PIXELS = 1;

Adafruit_NeoPixel strip(NUM_PIXELS, NEO_PIN, NEO_GRB + NEO_KHZ800);

int brightness = 0;
int fadeAmount = 5;

void setup() {
  strip.begin();
  strip.show(); // Initialize all pixels to 'off'
}

void loop() {
  // Set pixel 0 to Red with changing brightness
  strip.setPixelColor(0, strip.Color(brightness, 0, 0));
  strip.show();

  // Adjust brightness
  brightness = brightness + fadeAmount;

  // Reverse fade direction at limits
  if (brightness <= 0 || brightness >= 255) {
    fadeAmount = -fadeAmount;
  }

  delay(30); // Control breath speed
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **WS2812B NeoPixel** onto the canvas.
2. Connect NeoPixel: **VCC** to **3V3**, **DIN** to **GP16**, **GND** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Observe the NeoPixel LED fading in a breathing pattern.

## Expected Output

Terminal:
```
Simulation active. Driving WS2812B NeoPixel on GP16.
```

## Expected Canvas Behavior
| Brightness Index | NeoPixel LED Color | Intensity |
| --- | --- | --- |
| 0 | OFF (Black) | 0% |
| 128 | Red | 50% |
| 255 | Red | 100% |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Adafruit_NeoPixel strip(...)` | Configures the NeoPixel object to target 1 LED on GP16 using 800 kHz protocol speed. |
| `strip.setPixelColor(0, ...)` | Sets the color values (Red, Green, Blue) for pixel index 0 in the local buffer array. |
| `strip.show()` | Transmits the data buffer to the NeoPixel chip over the single-wire data line. |

## Hardware & Safety Concept: NeoPixel Signal Timing
WS2812B NeoPixels use a high-speed, single-wire communication protocol where data is transmitted via precise pulse widths. A logic `0` requires a high pulse of 0.4 microseconds followed by a low pulse of 0.85 microseconds, while a logic `1` requires a high pulse of 0.8 microseconds and a low pulse of 0.45 microseconds. Because these timings have strict tolerances (nanoseconds), the driver library disables interrupts during transmission to prevent timing disruptions.

## Try This! (Challenges)
1. **Green Breathing**: Change the active color parameter from Red to Green.
2. **Rainbow Cycle**: Write code to fade from Red to Green, then Green to Blue, then Blue back to Red.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| NeoPixel displays wrong colors (e.g. Green instead of Red) | Wrong color order configuration | Hobby NeoPixels can be RGB or GRB. Adjust the configuration flag in setup (e.g. swap `NEO_GRB` with `NEO_RGB`). |

## Mode Notes
This basic addressable LED project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [65 - Pico NeoPixel Chase](65-pico-neopixel-chase.md)
- [70 - Pico NeoPixel Mixer](../../intermediate/70-pico-neopixel-mixer.md)
