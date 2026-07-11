# 64 - NeoPixel Single LED Fade

Fade the brightness of a single addressable WS2812B NeoPixel LED in a breathing pattern using the VEGA ARIES v3 board.

## Goal
Learn how to configure the Adafruit NeoPixel library, transmit 24-bit color data packets over a single wire, and adjust color brightness levels in a loop-free C++ state machine.

## What You Will Build
A breathing green lighting effect where a single NeoPixel LED fades its intensity up from 0 to 255 (maximum) and back down to 0 continuously.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| WS2812B NeoPixel Module | `neopixel` | Yes (1 pixel) | Yes (1-pixel breakout) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NeoPixel | VCC | 3V3 | Red | Power supply |
| NeoPixel | DIN | GPIO 14 | Blue | Single-wire data line |
| NeoPixel | GND | GND | Black | Ground reference |

> **Wiring tip:** WS2812B NeoPixels use a single-wire control protocol. DIN connects to GPIO 14. Keep data wires short to reduce signal noise.

## Code
```cpp
#include <Adafruit_NeoPixel.h>

const int NEO_PIN = 14;
const int NUM_PIXELS = 1;

Adafruit_NeoPixel strip(NUM_PIXELS, NEO_PIN, NEO_GRB + NEO_KHZ800);

int brightness = 0;
int fadeAmount = 5;

void setup() {
  strip.begin();
  strip.show(); // Initialize the pixel to 'off'
}

void loop() {
  // Set pixel 0 to Green with changing brightness (Red=0, Green=brightness, Blue=0)
  strip.setPixelColor(0, strip.Color(0, brightness, 0));
  strip.show();

  brightness = brightness + fadeAmount;

  // Reverse fade direction at limits (0 and 255)
  if (brightness <= 0 || brightness >= 255) {
    fadeAmount = -fadeAmount;
  }

  delay(30); // Controls breathing speed (30 ms per step)
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** and **WS2812B NeoPixel** components onto the canvas.
2. Connect the NeoPixel: **VCC** to **3V3**, **DIN** to **GPIO 14**, and **GND** to **GND**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
NeoPixel active on GPIO 14.
```

## Expected Canvas Behavior
* The single NeoPixel widget fades in and out with a green breathing pattern (0% to 100% intensity).

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `#include <Adafruit_NeoPixel.h>` | Imports the NeoPixel control library. |
| `Adafruit_NeoPixel strip(...)` | Instantiates driver specifying 1 pixel, control pin 14, and GRB/800kHz protocol settings. |
| `strip.begin()` | Prepares the digital output pin for data transmission. |
| `strip.setPixelColor(0, ...)` | Sets the color values for pixel 0 in the local buffer (red=0, green=brightness, blue=0). |
| `strip.show()` | Writes the buffer data out to the physical shift registers inside the LED. |

## Hardware & Safety Concept
* **WS2812B Data Protocol**: WS2812B LEDs use a precise, high-speed, single-wire timing protocol (800 kHz). Data is sent as a series of 24-bit color codes (8 bits each for Green, Red, and Blue). Each LED consumes its 24 bits and forwards the rest down the strip.
* **Power Filtering**: Addressable LEDs draw rapid pulses of current when switching states, which can introduce electrical noise. Connecting a 100 µF capacitor across the power leads (VCC and GND) near the pixel helps stabilize voltage and prevent glitches.

## Try This! (Challenges)
1. **Red Fader**: Change the code so that the NeoPixel breathes Red instead of Green.
2. **Channel Cycle**: Modify the program to fade Green, then Blue, then Red in sequence. Use a state variable to track the active color.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| NeoPixel is lit with full bright white and does not fade | Data line signal error | Verify that the DIN pin is connected specifically to GPIO 14 and that the ground is shared. |
| Color flickering or random colors | Timing issue or noisy power | Keep data wires under 20 cm and add a 330 ohm resistor in series with the DIN line on physical circuits. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [65 - NeoPixel Strip Cycle Chase](65-neopixel-strip-cycle-chase.md)
- [69 - NeoPixel Color Slider Mixer](69-neopixel-color-slider-mixer.md)
