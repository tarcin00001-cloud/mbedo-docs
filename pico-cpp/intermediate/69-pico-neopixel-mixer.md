# 69 - Pico NeoPixel Mixer

Mix RGB color channels on an addressable NeoPixel LED using three analog potentiometer sliders.

## Goal
Learn how to read multiple analog inputs and map them to control the Red, Green, and Blue light intensities of a WS2812B NeoPixel.

## What You Will Build
An analog color mixer board:
- **Red Potentiometer (GP26)**: Controls Red intensity (0 to 255).
- **Green Potentiometer (GP27)**: Controls Green intensity (0 to 255).
- **Blue Potentiometer (GP28)**: Controls Blue intensity (0 to 255).
- **NeoPixel LED (GP16)**: Displays the blended color output.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Potentiometers | `potentiometer` | Yes (three potentiometers) | Yes |
| WS2812B NeoPixel | `neopixel` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Red Potentiometer | Wiper (Pin 2) | GP26 (ADC0) | Red level |
| Green Potentiometer| Wiper (Pin 2) | GP27 (ADC1) | Green level |
| Blue Potentiometer | Wiper (Pin 2) | GP28 (ADC2) | Blue level |
| NeoPixel | DIN | GP16 | Data line |
| All Pot & Pixels | VCC | 3.3V | Shared power |
| All Pot & Pixels | GND | GND | Common ground return |

## Code
```cpp
#include <Adafruit_NeoPixel.h>

const int RED_POT = 26;
const int GRN_POT = 27;
const int BLU_POT = 28;
const int NEO_PIN = 16;

Adafruit_NeoPixel strip(1, NEO_PIN, NEO_GRB + NEO_KHZ800);

void setup() {
  pinMode(RED_POT, INPUT);
  pinMode(GRN_POT, INPUT);
  pinMode(BLU_POT, INPUT);
  strip.begin();
  strip.show();
}

void loop() {
  // Read analog sliders (0 to 4095)
  int rVal = analogRead(RED_POT);
  int gVal = analogRead(GRN_POT);
  int bVal = analogRead(BLU_POT);

  // Map 12-bit inputs to 8-bit color intensities (0 to 255)
  int rIntensity = rVal / 16;
  int gIntensity = gVal / 16;
  int bIntensity = bVal / 16;

  // Output blended color to NeoPixel
  strip.setPixelColor(0, strip.Color(rIntensity, gIntensity, bIntensity));
  strip.show();

  delay(50); // High refresh rate
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **three Potentiometers**, and **WS2812B NeoPixel** onto the canvas.
2. Connect Potentiometers: Red to **GP26**, Green to **GP27**, Blue to **GP28**.
3. Connect NeoPixel: **DIN** to **GP16**. Connect all power and ground rails.
4. Paste code, select the interpreted mode, and click **Run**.
5. Adjust the three sliders on the canvas to mix custom colors (e.g. Red + Green = Yellow).

## Expected Output

Terminal:
```
Simulation active. Mixing analog inputs to GP16 NeoPixel.
```

## Expected Canvas Behavior
| Red Slider (GP26) | Green Slider (GP27) | Blue Slider (GP28) | NeoPixel LED Color |
| --- | --- | --- | --- |
| Max (4095) | Min (0) | Min (0) | Red |
| Max (4095) | Max (4095) | Min (0) | Yellow |
| Min (0) | Max (4095) | Max (4095) | Cyan |
| Max (4095) | Max (4095) | Max (4095) | White |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `strip.Color(rIntensity, gIntensity, bIntensity)` | Converts independent 8-bit R, G, B color values into a consolidated 32-bit color word. |

## Hardware & Safety Concept: Decoupling Power Lines
Potentiometer sliders generate high noise on analog voltage lines when adjusted. Because the ADC uses the Pico's 3.3V line as its reference voltage, noise on power rails degrades reading accuracy. To keep readings stable, place a **0.1uF ceramic capacitor** physically near the VCC pin of each potentiometer to filter out high-frequency noise.

## Try This! (Challenges)
1. **Brightness Master**: Add a fourth potentiometer on GP29. Use its value to scale the overall brightness of the NeoPixel using `strip.setBrightness()`.
2. **Reverse Mixer**: Make the NeoPixel shine white when all pots are at minimum, and dim to black when all are at maximum.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Colors flicker when sliders are static | ADC channel noise | Add a small noise gate in code (only update NeoPixel if raw pot value changes by more than 30 units). |

## Mode Notes
This multi-channel analog mapping project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [32 - Pico Potentiometer Dimmer](../../beginner/32-pico-potentiometer-dimmer.md)
- [64 - Pico NeoPixel Fade](64-pico-neopixel-fade.md)
