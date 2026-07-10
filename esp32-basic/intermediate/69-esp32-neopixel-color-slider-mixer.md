# 69 - ESP32 NeoPixel Color Slider Mixer

Mix custom colours on a NeoPixel strip in real time using three potentiometers dedicated to controlling the Red, Green, and Blue channels.

## Goal
Learn how to read multiple analog inputs concurrently, map their values to 8-bit brightness levels, and write the mixed colour data to a multi-LED NeoPixel strip.

## What You Will Build
Three potentiometers read analog voltages on GPIO 34 (Red), GPIO 35 (Green), and GPIO 36 (Blue). The code maps each input to a range of 0–255 and sets all 8 LEDs on a WS2812B NeoPixel strip to the resulting mixed colour.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| WS2812B NeoPixel Strip (8 LEDs) | `neopixel` | Yes | Yes |
| Potentiometers (3) | `potentiometer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NeoPixel Strip | VCC | 5V (Vin) | Red | Power supply |
| NeoPixel Strip | GND | GND | Black | Ground |
| NeoPixel Strip | DIN | GPIO5 | Orange | Data signal |
| Pot R (Red) | Wiper | GPIO34 | Yellow | Red intensity control |
| Pot R (Red) | VCC / GND | 3V3 / GND | Red / Black | Shared power rails |
| Pot G (Green) | Wiper | GPIO35 | Green | Green intensity control |
| Pot G (Green) | VCC / GND | 3V3 / GND | Red / Black | Shared power rails |
| Pot B (Blue) | Wiper | GPIO36 | Blue | Blue intensity control |
| Pot B (Blue) | VCC / GND | 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** All three potentiometers share the same 3V3 VCC rail and GND rail. Use GPIO 34, 35, and 36 because they are analog input channels connected to the ADC1 controller.

## Code
```cpp
// NeoPixel Color Slider Mixer
#include <Adafruit_NeoPixel.h>

#define LED_PIN 5
#define NUM_LEDS 8

const int POT_R = 34;
const int POT_G = 35;
const int POT_B = 36;

Adafruit_NeoPixel strip(NUM_LEDS, LED_PIN, NEO_GRB + NEO_KHZ800);

void setup() {
  Serial.begin(115200);
  
  strip.begin();
  strip.show(); // Initialize all pixels to 'off'
  
  Serial.println("NeoPixel Color Slider Mixer ready.");
}

void loop() {
  int rawR = analogRead(POT_R);
  int rawG = analogRead(POT_G);
  int rawB = analogRead(POT_B);
  
  // Map 12-bit ADC (0-4095) to 8-bit color space (0-255)
  int r = map(rawR, 0, 4095, 0, 255);
  int g = map(rawG, 0, 4095, 0, 255);
  int b = map(rawB, 0, 4095, 0, 255);
  
  // Clamp values for safety
  r = constrain(r, 0, 255);
  g = constrain(g, 0, 255);
  b = constrain(b, 0, 255);
  
  // Apply the color to all pixels
  for (int i = 0; i < NUM_LEDS; i++) {
    strip.setPixelColor(i, strip.Color(r, g, b));
  }
  strip.show();
  
  Serial.print("RGB Mixed: (");
  Serial.print(r); Serial.print(", ");
  Serial.print(g); Serial.print(", ");
  Serial.print(b); Serial.println(")");
  
  delay(100); // 10Hz refresh rate
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **NeoPixel (8 LEDs)**, and three **Potentiometers** onto the canvas.
2. Wire the potentiometer wipers to **GPIO34, 35, 36**. Wire the NeoPixel DIN to **GPIO5**.
3. Paste the code and click **Run**.
4. Adjust the three sliders on the canvas to mix custom colors (e.g. max R and B for purple).

## Expected Output
Serial Monitor:
```
NeoPixel Color Slider Mixer ready.
RGB Mixed: (0, 0, 0)
RGB Mixed: (255, 0, 255)
RGB Mixed: (128, 255, 64)
```

## Expected Canvas Behavior
* Adjusting the Pot R slider increases/decreases the red brightness of the NeoPixel strip.
* Adjusting Pot G and Pot B sliders changes green and blue intensities.
* The combined mix is reflected in real time on the virtual NeoPixel strip.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `analogRead(POT_R)` | Reads the analog voltage from the red-channel potentiometer. |
| `map(rawR, 0, 4095, 0, 255)` | Converts the 12-bit ADC value to an 8-bit PWM value. |
| `strip.Color(r, g, b)` | Packages the RGB values into a single 32-bit color format. |
| `strip.show()` | Writes the updated color data out to the strip over GPIO 5. |

## Hardware & Safety Concept: RGB Color Theory and Inrush Currents
White light is produced by mixing Red, Green, and Blue at equal full intensities. At full brightness, a strip of 8 LEDs consumes: 8 × 3 × 20 mA = 480 mA. A standard computer USB port cannot safely supply much more than 500mA without dropping voltage or triggering overcurrent protection. Keep the colors mixed or use a separate power source if using longer strips (e.g. 30+ LEDs).

## Try This! (Challenges)
1. **Master Dimmer**: Add a fourth potentiometer on GPIO 39 to act as a master brightness scaler.
2. **Preset Swapper**: Add a button that cycles through custom preset colors, bypassing the sliders when active.
3. **Color Transition**: Implement a smooth fade when moving from one slider configuration to another instead of instant updates.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Strip displays incorrect colors | Pin connections or RGB order swap | Change `NEO_GRB` to `NEO_RGB` in constructor if colors are swapped |
| Strip flickers or turns off | Insufficient power supply | Lower the maximum mapped brightness bounds (e.g. 0–150) |
| Slider adjustments feel lagged | Loop delay is too high | Lower the `delay()` time at the end of the loop |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [64 - ESP32 NeoPixel Single LED Colour Cycle](64-esp32-neopixel-single-led-colour-cycle.md)
- [65 - ESP32 NeoPixel Strip Chase Animation](65-esp32-neopixel-strip-chase-animation.md)
- [32 - ESP32 Potentiometer LED Dimmer](../beginner/32-esp32-potentiometer-led-dimmer.md)
