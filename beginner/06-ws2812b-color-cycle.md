# 06 - WS2812B Color Cycle

Cycle an addressable RGB LED through red, green, and blue using the Adafruit NeoPixel library.

## Goal
Learn how to declare, configure, and update a WS2812B (NeoPixel) addressable LED using standard library syntax.

## What You Will Build
A single WS2812B addressable LED connected to pin D6 cycles through primary colors (Red -> Green -> Blue) at 500 ms intervals.

**Why D6?** Digital pin D6 is a high-speed data line capable of pushing the high-frequency control signals needed by WS2812B pixels.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| WS2812B LED | `ws2812b` | Yes | Yes |
| 470 ohm resistor | `resistor` | Optional | Yes (recommended on data line) |

## Wiring
| WS2812B Pin | Arduino Pin | Notes |
| --- | --- | --- |
| DIN (Data Input) | D6 | High-speed data line |
| VCC | 5V | 5V Power supply |
| GND | GND | Ground connection |

> On physical hardware, a **470 ohm resistor** should be wired in series on the `DIN` line to prevent voltage spikes from damaging the first pixel's controller.

## Code
```cpp
#include <Adafruit_NeoPixel.h>

const int DATA_PIN  = 6;
const int NUM_LEDS  = 1;

// Define the NeoPixel object
Adafruit_NeoPixel pixels(NUM_LEDS, DATA_PIN, NEO_GRB + NEO_KHZ800);

void setup() {
  // Initialize the library
  pixels.begin();
  
  Serial.begin(9600);
  Serial.println("WS2812B Color Cycle ready");
}

void loop() {
  // Red
  pixels.setPixelColor(0, pixels.Color(255, 0, 0));
  pixels.show();
  Serial.println("Red");
  delay(500);

  // Green
  pixels.setPixelColor(0, pixels.Color(0, 255, 0));
  pixels.show();
  Serial.println("Green");
  delay(500);

  // Blue
  pixels.setPixelColor(0, pixels.Color(0, 0, 255));
  pixels.show();
  Serial.println("Blue");
  delay(500);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** onto the canvas.
2. Drag **WS2812B LED** onto the canvas.
3. Connect WS2812B **DIN** to Arduino **D6**.
4. Connect WS2812B **VCC** to Arduino **5V**.
5. Connect WS2812B **GND** to Arduino **GND**.
6. Paste the code into the editor.
7. Use the default Arduino interpreted runtime.
8. Click **Run**.

## Expected Output

Serial Monitor:
```
WS2812B Color Cycle ready
Red
Green
Blue
...
```

### Expected Canvas Behavior

| Step | Function Called | Color Argument (R,G,B) | Resulting Color |
| --- | --- | --- | --- |
| 1 | `setPixelColor(0, ...)` | `(255, 0, 0)` | Red |
| 2 | `setPixelColor(0, ...)` | `(0, 255, 0)` | Green |
| 3 | `setPixelColor(0, ...)` | `(0, 0, 255)` | Blue |

The addressable LED pixel on the canvas cycles colors quickly.

## Code Walkthrough

| Line | What It Does |
| --- | --- |
| `#include <Adafruit_NeoPixel.h>` | Imports the NeoPixel library shim to communicate with addressable LED controllers. |
| `Adafruit_NeoPixel pixels(...)` | Instantiates a controller object named `pixels` pointing to Pin D6. |
| `pixels.begin()` | Prepares the digital output pin for data transmission. |
| `pixels.setPixelColor(index, color)` | Buffers a color for a specific pixel index (starts at 0). This does not light the LED yet. |
| `pixels.show()` | Transmits the buffered color data down the wire, updating the active LEDs. |

## Hardware & Safety Concept: Addressable LEDs (Smart Pixels)
Standard RGB LEDs require three dedicated control lines from the Arduino. A **WS2812B** (NeoPixel) has an integrated microcontroller chip (WS2811) built directly into the LED package. 
- It uses a single high-speed data wire to receive 24-bit color packets.
- When multiple pixels are daisy-chained, each pixel strips the first 24 bits of color from the data stream and passes the rest to the next pixel. This allows you to control hundreds of pixels independently using only 1 Arduino data pin.

## Try This! (Challenges)
1. **Change Colors**: Modify the RGB values to display Yellow (255, 255, 0) and Orange (255, 128, 0).
2. **Speed Cycle**: Decrease the delay from `500` to `150`. Notice how much faster the color transitions occur.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED remains dark | Missing power or ground lines | Ensure WS2812B VCC connects to 5V and GND connects to GND. Addressable LEDs cannot run on data signals alone. |
| Colors show incorrect shades | Wrong data pin configured | Verify the data pin is wired to D6, matching the `DATA_PIN = 6` constant. |

## Mode Notes
The `Adafruit_NeoPixel` library is shimmed in the MbedO interpreter. Fixed-index commands such as `pixels.setPixelColor(0, ...)` and `pixels.show()` are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [04 - RGB Color Mixer](04-rgb-color-mixer.md)
- [05 - Traffic Light Sequence](05-traffic-light-sequence.md)
