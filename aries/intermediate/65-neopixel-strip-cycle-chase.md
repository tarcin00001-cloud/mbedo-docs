# 65 - NeoPixel Strip Cycle Chase

Create a running light chase animation on a 4-pixel WS2812B NeoPixel strip using the VEGA ARIES v3 board without using loops in C++ code.

## Goal
Learn how to program sequential addressable LED strip animations by using state variables and manual pixel buffer clearing to comply with interpreted mode constraints.

## What You Will Build
A theater-style running light chase where a single blue dot travels along a 4-LED strip from pixel 0 to 3, resetting and repeating continuously.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| WS2812B NeoPixel Strip | `neopixel` | Yes (4 pixels) | Yes (4-pixel strip) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NeoPixel Strip | VCC | 3V3 | Red | Power supply |
| NeoPixel Strip | DIN | GPIO 14 | Blue | Data input line |
| NeoPixel Strip | GND | GND | Black | Ground reference |

> **Wiring tip:** Standard NeoPixel strips require a single signal connection to GPIO 14. Insulate any exposed strip connections to prevent short circuits.

## Code
```cpp
#include <Adafruit_NeoPixel.h>

const int NEO_PIN = 14;
const int NUM_PIXELS = 4;

Adafruit_NeoPixel strip(NUM_PIXELS, NEO_PIN, NEO_GRB + NEO_KHZ800);

int activePixel = 0;

void setup() {
  strip.begin();
  strip.show(); // Initialize the strip to all pixels 'off'
}

void loop() {
  // Clear all pixels individually without using loops (for/while are not allowed)
  strip.setPixelColor(0, strip.Color(0, 0, 0));
  strip.setPixelColor(1, strip.Color(0, 0, 0));
  strip.setPixelColor(2, strip.Color(0, 0, 0));
  strip.setPixelColor(3, strip.Color(0, 0, 0));

  // Set the currently active pixel to Blue
  strip.setPixelColor(activePixel, strip.Color(0, 0, 150));
  strip.show();

  // Move to the next pixel index
  activePixel = activePixel + 1;

  // Wrap around back to the beginning of the strip
  if (activePixel >= 4) {
    activePixel = 0;
  }

  delay(250); // Speed of the chase animation (250 ms per step)
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** and **WS2812B NeoPixel Strip** components onto the canvas. (Verify strip has exactly 4 pixels).
2. Wire the strip: **VCC** to **3V3**, **DIN** to **GPIO 14**, and **GND** to **GND**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
NeoPixel Chase Started on GPIO 14.
```

## Expected Canvas Behavior
* A single blue LED runs down the 4-pixel strip from index 0 to 3, wraps back to 0, and repeats every 250 ms.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Adafruit_NeoPixel strip(...)` | Configures a 4-pixel strip on GPIO pin 14. |
| `strip.setPixelColor(0, ...)` | Manually clears each pixel in the buffer without needing a forbidden for-loop. |
| `strip.setPixelColor(activePixel, ...)` | Lights the target index to blue. |
| `activePixel = activePixel + 1` | Increments the pointer state variable. |
| `if (activePixel >= 4)` | Limits the index range to indices 0, 1, 2, and 3. |

## Hardware & Safety Concept
* **Timing Constraints**: WS2812B LEDs require very precise clock timings. A logic '0' pulse is 0.35 µs high / 0.8 µs low, and a logic '1' is 0.7 µs high / 0.6 µs low. To achieve this on a 100 MHz RISC-V SoC, the BSP handles these timing loops at assembly level, which are abstracted away by the Adafruit library.
* **Power Consumption**: Each WS2812B pixel can draw up to 60 mA at full brightness white (20 mA per color channel). A 4-pixel strip draws up to 240 mA, which the ARIES board regulator can supply safely. However, strips longer than 8 pixels should always have an external power supply to prevent regulator overheating.

## Try This! (Challenges)
1. **Dual Chaser**: Modify the logic to have two blue pixels active at once, separated by one off-pixel (e.g. index 0 and index 2 turn on together).
2. **Dynamic Colors**: Cycle the color of the chase pixel between Red, Green, and Blue on successive passes of the strip.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Only the first pixel lights up | Strip size mismatch in code | Ensure `NUM_PIXELS` is set to 4 in code, and the canvas strip is set to 4 LEDs. |
| Strange colors on some pixels | Poor ground connection | Check that the ground wire from the NeoPixel strip to the ARIES GND is connected securely. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [64 - NeoPixel Single LED Fade](64-neopixel-single-led-fade.md)
- [69 - NeoPixel Color Slider Mixer](69-neopixel-color-slider-mixer.md)
