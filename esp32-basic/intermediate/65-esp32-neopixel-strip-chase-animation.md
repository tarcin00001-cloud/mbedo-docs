# 65 - ESP32 NeoPixel Strip Chase Animation

Animate a "chase" pattern across an 8-LED NeoPixel strip — a single lit pixel travels the length of the strip, leaving a fading tail — the classic Knight Rider / LED chaser effect in full colour.

## Goal
Learn how to control multiple NeoPixel LEDs simultaneously, implement a colour-fading tail by scaling brightness, and change the chase colour each cycle using the HSV colour wheel.

## What You Will Build
An 8-LED WS2812B NeoPixel strip on GPIO 5. A bright head pixel chases across all 8 LEDs with a 3-pixel dimming tail. Each full cycle through the strip uses a new hue, creating a rainbow-chasing effect.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| WS2812B NeoPixel Strip (8 LEDs) | `neopixel` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NeoPixel Strip | VCC (5V) | 5V (Vin) | Red | Strip supply — 8 LEDs at 60 mA each = up to 480 mA |
| NeoPixel Strip | GND | GND | Black | Common ground |
| NeoPixel Strip | DIN (Data In) | GPIO5 via 470 Ω | Orange | Serial data — series resistor dampens reflections |

> **Wiring tip:** 8 WS2812B LEDs at full white draw up to 480 mA. At the brightness level used in this project (~40/255), actual current is ~90 mA — within USB capability. Add a 470 µF electrolytic capacitor across the 5 V and GND rails near the LED strip to handle current spikes when many LEDs change simultaneously. Connect the capacitor with correct polarity (positive to 5 V).

## Code
```cpp
// NeoPixel Strip Chase Animation — fading tail + rainbow hue
#include <Adafruit_NeoPixel.h>

#define LED_PIN    5
#define NUM_LEDS   8
#define BRIGHTNESS 40   // Master brightness (0–255)
#define TAIL_LEN   3    // Number of tail pixels behind the head

Adafruit_NeoPixel strip(NUM_LEDS, LED_PIN, NEO_GRB + NEO_KHZ800);

uint16_t cycleHue = 0;   // Hue changes on each complete chase cycle

uint32_t dimColour(uint32_t colour, uint8_t factor) {
  // Scale each R, G, B channel by factor/255
  uint8_t r = (uint8_t)((colour >> 16) & 0xFF) * factor / 255;
  uint8_t g = (uint8_t)((colour >> 8)  & 0xFF) * factor / 255;
  uint8_t b = (uint8_t)((colour)       & 0xFF) * factor / 255;
  return strip.Color(r, g, b);
}

void setup() {
  strip.begin();
  strip.setBrightness(BRIGHTNESS);
  strip.show();
  Serial.begin(115200);
  Serial.println("NeoPixel Chase Animation ready.");
}

void loop() {
  uint32_t headColour = strip.gamma32(strip.ColorHSV(cycleHue, 255, 255));

  for (int head = 0; head < NUM_LEDS; head++) {
    strip.clear();   // All off

    // Draw head
    strip.setPixelColor(head, headColour);

    // Draw tail with decreasing brightness
    for (int t = 1; t <= TAIL_LEN; t++) {
      int tailPos = head - t;
      if (tailPos >= 0) {
        uint8_t dimFactor = 255 / (t + 1);   // 127, 85, 63 for t=1,2,3
        strip.setPixelColor(tailPos, dimColour(headColour, dimFactor));
      }
    }

    strip.show();

    Serial.print("Head at pixel "); Serial.print(head);
    Serial.print("  Hue: "); Serial.println(cycleHue);

    delay(60);   // 60 ms per step = ~480 ms per full 8-pixel chase
  }

  // Advance hue for the next chase cycle
  cycleHue = (cycleHue + 8192) % 65536;   // 8 steps across the colour wheel
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **NeoPixel** (set count = 8) onto the canvas.
2. Connect NeoPixel **DIN** to **GPIO5**.
3. Paste the code and click **Run**.
4. Observe the chase pattern sweeping across the strip, changing colour each pass.

## Expected Output
Serial Monitor:
```
NeoPixel Chase Animation ready.
Head at pixel 0  Hue: 0
Head at pixel 1  Hue: 0
Head at pixel 7  Hue: 0
Head at pixel 0  Hue: 8192
Head at pixel 0  Hue: 16384
```

## Expected Canvas Behavior
* A bright leading pixel sweeps from LED 0 to LED 7 with a dimming 3-pixel tail.
* Each new pass uses a different colour (red → yellow → green → cyan → blue → magenta → back to red over 8 passes).
* All other pixels are off — only the head and tail are lit.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `strip.clear()` | Sets all pixels in the buffer to black (off) at the start of each head position. |
| `strip.setPixelColor(head, headColour)` | Sets the head pixel to full brightness in the current hue. |
| `uint8_t dimFactor = 255 / (t + 1)` | Computes tail brightness: t=1 → 127 (50%), t=2 → 85 (33%), t=3 → 63 (25%). |
| `dimColour(headColour, dimFactor)` | Scales all three RGB channels proportionally by the dim factor. |
| `cycleHue = (cycleHue + 8192) % 65536` | Advances the hue by 1/8 of the wheel per cycle (8 × 8192 = 65536 = full circle). |

## Hardware & Safety Concept: WS2812B Cascaded Addressing
In a NeoPixel strip, LEDs are wired in a cascade — each LED's DOUT pin connects to the next LED's DIN. The first LED receives all data, consumes its first 24 bits (for itself), then passes the remaining bits downstream. The second LED consumes its 24 bits and passes the rest onward, and so on. The Adafruit NeoPixel library builds the complete bitstream for all LEDs in RAM, then sends the entire stream once per `show()` call. This means updating a single pixel still requires transmitting data for all LEDs in the chain — but at 800 kHz the full 192-bit stream for 8 LEDs takes only 240 µs, far faster than the 50 µs LED reset gap required between transmissions.

## Try This! (Challenges)
1. **Bidirectional bounce**: After the head reaches pixel 7, reverse direction (sweep 7→0) without changing hue.
2. **Two chasers**: Run two head positions simultaneously (offset by `NUM_LEDS / 2`) using complementary hues.
3. **Speed control**: Read a potentiometer on GPIO 34 and map it to the step delay (20–200 ms) for real-time speed control.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| All 8 LEDs light at once | Missing `strip.clear()` before setting head | Confirm `strip.clear()` is the first call inside the inner for loop |
| Tail not visible | `TAIL_LEN = 0` or tail pixel index negative | Verify `TAIL_LEN = 3`; the boundary check `if (tailPos >= 0)` handles edge cases |
| Strip flickers | 5 V power sag from USB | Add a 470 µF electrolytic capacitor across the strip's 5 V and GND pads |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [64 - ESP32 NeoPixel Single LED Colour Cycle](64-esp32-neopixel-single-led-colour-cycle.md)
- [77 - ESP32 NeoPixel Sensor Visualiser](77-esp32-neopixel-sensor-visualiser.md)
- [04 - ESP32 Multiple LED Chase](../beginner/04-esp32-multiple-led-chase.md)
