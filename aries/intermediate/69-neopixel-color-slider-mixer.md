# 69 - NeoPixel Color Slider Mixer

Mix and blend colors on a single WS2812B NeoPixel LED using an analog potentiometer and the VEGA ARIES v3 board.

## Goal
Learn how to read analog inputs, implement additive color mixing math, and drive addressable NeoPixel LEDs dynamically in a loop-free C++ state machine.

## What You Will Build
An interactive color mixer console where rotating the potentiometer dial blends the colors on the NeoPixel LED smoothly across the visible spectrum (from Red to Green to Blue and back to Red).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| WS2812B NeoPixel Module | `neopixel` | Yes (1 pixel) | Yes (1-pixel breakout) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Pin 1 (VCC) | 3V3 | Red | Analog voltage supply |
| Potentiometer | Pin 2 (Wiper) | ADC0 (GP26) | White | Analog output signal |
| Potentiometer | Pin 3 (GND) | GND | Black | Ground reference |
| NeoPixel | VCC | 3V3 | Red | Power supply |
| NeoPixel | DIN | GPIO 14 | Blue | Single-wire data line |
| NeoPixel | GND | GND | Black | Ground reference |

> **Wiring tip:** Share the GND connection on a breadboard. Ensure the NeoPixel is connected to 3.3V power to match the ARIES SoC logic levels.

## Code
```cpp
#include <Adafruit_NeoPixel.h>

const int POT_PIN = 26; // ADC0 is GP26
const int NEO_PIN = 14;
const int NUM_PIXELS = 1;

Adafruit_NeoPixel strip(NUM_PIXELS, NEO_PIN, NEO_GRB + NEO_KHZ800);

void setup() {
  strip.begin();
  strip.show(); // Initialize the pixel to 'off'
}

void loop() {
  int potVal = analogRead(POT_PIN);

  // Divide the potentiometer range (0-4095) into three color zones:
  // Zone 1 (0 to 1365): Fade from Red to Green
  // Zone 2 (1366 to 2730): Fade from Green to Blue
  // Zone 3 (2731 to 4095): Fade from Blue to Red
  int r = 0;
  int g = 0;
  int b = 0;

  if (potVal < 1365) {
    r = (1365 - potVal) * 255 / 1365;
    g = potVal * 255 / 1365;
    b = 0;
  } else if (potVal < 2730) {
    r = 0;
    g = (2730 - potVal) * 255 / 1365;
    b = (potVal - 1365) * 255 / 1365;
  } else {
    r = (potVal - 2730) * 255 / 1365;
    g = 0;
    b = (4095 - potVal) * 255 / 1365;
  }

  strip.setPixelColor(0, strip.Color(r, g, b));
  strip.show();

  delay(50); // Polling delay (50 ms)
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **Potentiometer**, and **WS2812B NeoPixel** components onto the canvas.
2. Connect the Potentiometer: **Pin 1** to **3V3**, **Wiper** to **ADC0 (GP26)**, and **Pin 3** to **GND**.
3. Connect the NeoPixel: **VCC** to **3V3**, **DIN** to **GPIO 14**, and **GND** to **GND**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
ADC: 0, RGB: (255, 0, 0)
ADC: 2048, RGB: (0, 127, 127)
```

## Expected Canvas Behavior
* Adjusting the potentiometer dial sweeps the NeoPixel color. It goes from solid Red at 0, through Orange/Yellow/Green, then Cyan/Blue, and finally Purple/Magenta back to Red at 4095.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `analogRead(POT_PIN)` | Reads potentiometer value (0 to 4095). |
| `potVal < 1365` | Evaluates if the slider is in the first third of its travel. |
| `(1365 - potVal) * 255 / 1365` | Calculates inverse fading component for the Red channel. |
| `strip.setPixelColor(...)` | Applies the blended RGB values to pixel index 0. |
| `strip.show()` | Writes the 24-bit data packet to the LED module. |

## Hardware & Safety Concept
* **Color Mixing (Additive Color)**: Blending different amounts of Red, Green, and Blue light creates colors (additive color model). Blending full Red and Green gives Yellow; Green and Blue gives Cyan; Blue and Red gives Magenta. Blending all three gives White.
* **Ground Loops**: Addressable LEDs draw current in rapid PWM pulses inside their packaging, which can cause minor ground voltage shifts. Connecting the potentiometer ground directly to the ARIES board's ground pin (rather than sharing the LED ground line) helps ensure clean, stable analog readings.

## Try This! (Challenges)
1. **Brightness Scalar**: Add logic to scale down the maximum color intensity from 255 to 100 to reduce the NeoPixel's current consumption.
2. **Dynamic Pulse**: Make the NeoPixel pulse (breathe) in brightness while displaying the color mapped by the potentiometer.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| NeoPixel does not light up or stays red | Mismatched pin wiring | Check that the data pin connects to GPIO 14 and the pot wiper is connected to GP26. |
| Color jumps instead of blending | Math overflow or integer rounding | Ensure calculation order keeps multiplications before divisions (e.g., `(potVal * 255) / 1365`) to prevent early division to 0. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [64 - NeoPixel Single LED Fade](64-neopixel-single-led-fade.md)
- [67 - Potentiometer Speed DC Motor (with LCD)](67-potentiometer-speed-dc-motor.md)
