# 64 - ESP32 NeoPixel Single LED Colour Cycle

Drive a single WS2812B NeoPixel RGB LED through a full colour wheel cycle using the Adafruit NeoPixel library and the ESP32's RMT peripheral for precise timing.

## Goal
Learn how the WS2812B addressable LED protocol works, how to send colour data using the Adafruit NeoPixel library, and how to smoothly cycle through the full colour spectrum using a hue-to-RGB conversion.

## What You Will Build
A single WS2812B NeoPixel LED connected to GPIO 5. The code cycles the LED through the full 360° hue wheel — red → yellow → green → cyan → blue → magenta → red — in smooth 1° steps.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| WS2812B NeoPixel LED (single or strip) | `neopixel` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| WS2812B NeoPixel | VCC (5V) | 5V (Vin) | Red | NeoPixel requires 5 V supply |
| WS2812B NeoPixel | GND | GND | Black | Common ground |
| WS2812B NeoPixel | DIN (Data In) | GPIO5 | Orange | Serial data signal |

> **Wiring tip:** Add a 300–500 Ω resistor in series on the DIN wire close to the LED to protect against signal reflections. Add a 100–1000 µF capacitor across VCC and GND to handle inrush current. Never power NeoPixels from the ESP32's 3V3 regulator — even a single LED at full white draws 60 mA, exceeding the regulator's safe output. Use the 5 V (Vin) pin from USB power.

## Code
```cpp
// NeoPixel Single LED Full Colour Cycle
#include <Adafruit_NeoPixel.h>

#define LED_PIN    5
#define NUM_LEDS   1
#define BRIGHTNESS 60   // 0–255; keep low for long chains (power budget)

Adafruit_NeoPixel strip(NUM_LEDS, LED_PIN, NEO_GRB + NEO_KHZ800);

// Convert hue (0–65535) to RGB using Adafruit's colourHSV helper
uint32_t hueToColour(uint16_t hue) {
  return strip.gamma32(strip.ColorHSV(hue, 255, BRIGHTNESS));
}

void setup() {
  strip.begin();
  strip.show();   // Initialise to off
  Serial.begin(115200);
  Serial.println("NeoPixel Colour Cycle ready.");
}

void loop() {
  // Full colour wheel: 0–65535 = 360°
  for (uint16_t hue = 0; hue < 65536; hue += 256) {
    strip.setPixelColor(0, hueToColour(hue));
    strip.show();

    if (hue % 8192 == 0) {
      Serial.print("Hue: "); Serial.print(hue);
      Serial.print(" / 65535  (~");
      Serial.print(hue * 360 / 65535);
      Serial.println("°)");
    }

    delay(5);   // 5 ms per step × 256 steps = ~1.28 s per full cycle
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **NeoPixel** (set count = 1) onto the canvas.
2. Connect NeoPixel **DIN** to **GPIO5**.
3. Paste the code and click **Run**.
4. Watch the NeoPixel widget cycle through all colours smoothly.

## Expected Output
Serial Monitor:
```
NeoPixel Colour Cycle ready.
Hue: 0 / 65535  (~0°)       ← Red
Hue: 8192 / 65535  (~45°)   ← Orange
Hue: 16384 / 65535  (~90°)  ← Yellow-Green
Hue: 32768 / 65535  (~180°) ← Cyan
Hue: 49152 / 65535  (~270°) ← Blue
```

## Expected Canvas Behavior
* NeoPixel widget starts red, smoothly transitions through yellow, green, cyan, blue, magenta, and back to red.
* One full cycle completes in approximately 1.3 seconds.
* Cycle repeats indefinitely.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `Adafruit_NeoPixel strip(NUM_LEDS, LED_PIN, NEO_GRB + NEO_KHZ800)` | Declares the strip object: 1 LED on GPIO 5, GRB order, 800 kHz data rate. |
| `strip.begin()` | Initialises the RMT peripheral and configures GPIO 5 as NeoPixel data output. |
| `strip.ColorHSV(hue, 255, BRIGHTNESS)` | Converts a 16-bit hue (0–65535), full saturation (255), and brightness to a 32-bit RGB value. |
| `strip.gamma32(...)` | Applies gamma correction — makes the colour ramp perceptually linear (colours look more uniform). |
| `strip.setPixelColor(0, colour)` | Loads the colour into pixel 0's buffer. |
| `strip.show()` | Sends the buffer to the LED via the RMT peripheral. |

## Hardware & Safety Concept: WS2812B Addressable LEDs
The WS2812B integrates an RGB LED, three current-limiting resistors, and a driver IC in a single 5×5 mm package. It uses a single-wire serial protocol at 800 kHz — each bit is encoded as a specific HIGH/LOW pulse width (0-bit: 350 ns HIGH / 900 ns LOW; 1-bit: 900 ns HIGH / 350 ns LOW). The ESP32's **RMT (Remote Control Transceiver) peripheral** is ideal for generating these precise sub-microsecond pulses in hardware without timing interference from the OS or other interrupts. Each LED in a chain receives 24 bits (8R + 8G + 8B), strips the first 24 bits for itself, and passes the rest downstream — enabling thousands of individually addressable LEDs on a single GPIO wire. Full white at 100% brightness draws 60 mA per LED; at 60/255 brightness it draws ~14 mA.

## Try This! (Challenges)
1. **Breathing effect**: Keep hue fixed at red (hue=0) and cycle BRIGHTNESS from 0 to 60 and back to simulate a breathing pulse.
2. **Random colour**: On each loop iteration, pick a random hue with `random(65536)` for a disco-light effect.
3. **Button hue lock**: Add a button on GPIO 4 — pressing it freezes the colour at the current hue until pressed again.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays off | DIN wire not connected or missing `strip.show()` | Verify GPIO 5 → DIN and confirm `strip.show()` is called after `setPixelColor` |
| LED flickers randomly | Power supply cannot supply enough current | Use an external 5 V supply; add a 470 µF cap across 5 V and GND |
| Colours look wrong (RGB instead of GRB) | Wrong pixel order specified | Change `NEO_GRB` to `NEO_RGB` in the strip constructor |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [65 - ESP32 NeoPixel Strip Chase Animation](65-esp32-neopixel-strip-chase-animation.md)
- [10 - ESP32 RGB LED Colour Cycle](../beginner/10-esp32-rgb-led-cycle.md)
- [77 - ESP32 NeoPixel Sensor Visualiser](77-esp32-neopixel-sensor-visualiser.md)
