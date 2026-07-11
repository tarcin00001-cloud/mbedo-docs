# 190 - ESP32 RMT Hardware NeoPixel Driver

Build a high-performance LED strip controller on the ESP32 that configures the internal Remote Control (RMT) hardware peripheral to generate high-speed WS2812B timing waveforms, driving a 16-pixel NeoPixel strip.

## Goal
Learn how to use the ESP32 RMT peripheral in transmitter mode to generate sub-microsecond waveforms, understand WS2812B protocols, and map colors.

## What You Will Build
A WS2812B NeoPixel strip (16 pixels) is connected to GPIO 13. A potentiometer on GPIO 34 adjusts the color hue. The ESP32 does not use standard software bit-banging. Instead, the hardware RMT transmitter generates the exact 800 kHz WS2812B pulses (400 ns/850 ns durations) in the background.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| WS2812B NeoPixel Strip (16 LEDs) | `neopixel` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |
| External 5V Power Supply | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NeoPixel Strip | DIN (Data In) | GPIO13 | Green | RMT Output pin |
| NeoPixel Strip | 5V / GND | 5V / GND | Red / Black | Power rails |
| Potentiometer | Wiper | GPIO34 | Yellow | Hue adjustment wiper |
| Shared power | VCC / GND | 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** WS2812B data lines require precise timings. Connect the data pin directly to GPIO 13. Power the strip from the 5V Vin rail.

## Code
```cpp
// RMT Hardware NeoPixel Driver (Fast State Outputs)
#include <Arduino.h>
#include "driver/rmt.h"

const int POT_PIN = 34;
#define NEOPIXEL_PIN 13
#define NUM_LEDS     16

#define RMT_TX_CHANNEL RMT_CHANNEL_0

// WS2812B Protocol Timings (in ticks at 40 MHz)
// 1 tick = 25 ns (80 MHz / 2)
// Bit 0: 350 ns HIGH (14 ticks), 800 ns LOW (32 ticks)
// Bit 1: 700 ns HIGH (28 ticks), 600 ns LOW (24 ticks)
const rmt_item32_t bit0 = {{{ 14, 1, 32, 0 }}}; 
const rmt_item32_t bit1 = {{{ 28, 1, 24, 0 }}}; 

// Array of RMT items (24 bits per pixel: 8 Green, 8 Red, 8 Blue)
const int ITEMS_PER_LED = 24;
const int TOTAL_ITEMS = NUM_LEDS * ITEMS_PER_LED;
rmt_item32_t rmtBuffer[TOTAL_ITEMS];

struct RGB {
  uint8_t r;
  uint8_t g;
  uint8_t b;
};

void initRMTTx() {
  Serial.println("Configuring RMT transmitter for WS2812B...");
  
  // 1. Configure RMT Transmitter parameters
  rmt_config_t rmt_tx_config;
  rmt_tx_config.rmt_mode = RMT_MODE_TX;
  rmt_tx_config.channel = RMT_TX_CHANNEL;
  rmt_tx_config.gpio_num = (gpio_num_t)NEOPIXEL_PIN;
  
  // Clock divider: 2. Base clock = 80 MHz -> 40 MHz tick rate (1 tick = 25 ns)
  rmt_tx_config.clk_div = 2; 
  
  rmt_tx_config.mem_block_num = 1;
  rmt_tx_config.tx_config.loop_en = false;
  rmt_tx_config.tx_config.carrier_en = false;
  rmt_tx_config.tx_config.idle_output_en = true;
  rmt_tx_config.tx_config.idle_level = RMT_IDLE_LEVEL_LOW;
  
  // 2. Initialize RMT channel
  rmt_config(&rmt_tx_config);
  
  // 3. Install RMT driver
  rmt_driver_install(RMT_TX_CHANNEL, 0, 0);
  
  Serial.println("RMT NeoPixel driver online.");
}

// Convert Color Hue to RGB
RGB hueToRGB(float hue) {
  float r = 0, g = 0, b = 0;
  int i = floor(hue * 6);
  float f = hue * 6 - i;
  float p = 0;
  float q = 1.0 - f;
  float t = f;
  
  switch (i % 6) {
    case 0: r = 1; g = t; b = p; break;
    case 1: r = q; g = 1; b = p; break;
    case 2: r = p; g = 1; b = t; break;
    case 3: r = p; g = q; b = 1; break;
    case 4: r = t; g = p; b = 1; break;
    case 5: r = 1; g = p; b = q; break;
  }
  
  // Scale brightness to 30% to protect power rails
  return { (uint8_t)(r * 80), (uint8_t)(g * 80), (uint8_t)(b * 80) };
}

// Encode RGB values into RMT pulse items
void updateStripColor(RGB color) {
  int itemIdx = 0;
  
  for (int led = 0; led < NUM_LEDS; led++) {
    // WS2812B uses GRB format
    uint32_t grbVal = (color.g << 16) | (color.r << 8) | color.b;
    
    // Convert 24 bits to RMT items (MSB first)
    for (int bit = 23; bit >= 0; bit--) {
      bool isOne = (grbVal & (1 << bit)) != 0;
      rmtBuffer[itemIdx++] = isOne ? bit1 : bit0;
    }
  }
  
  // 4. Stream frame buffer to NeoPixels using RMT transmitter
  // true: Wait for transmission to finish before returning
  rmt_write_items(RMT_TX_CHANNEL, rmtBuffer, TOTAL_ITEMS, true);
  
  // Send reset signal (keep line LOW for > 50 us)
  delayMicroseconds(60); 
}

void setup() {
  Serial.begin(115200);
  pinMode(POT_PIN, INPUT);
  
  initRMTTx();
}

void loop() {
  // Read potentiometer and map to hue (0.0 to 1.0)
  int potRaw = analogRead(POT_PIN);
  float hue = (float)potRaw / 4095.0;
  
  RGB activeColor = hueToRGB(hue);
  
  updateStripColor(activeColor);
  
  static unsigned long lastPrint = 0;
  if (millis() - lastPrint >= 500) {
    Serial.print("Pot: "); Serial.print(potRaw);
    Serial.print(" | Hue: "); Serial.print(hue, 2);
    Serial.print(" | RGB: (");
    Serial.print(activeColor.r); Serial.print(",");
    Serial.print(activeColor.g); Serial.print(",");
    Serial.print(activeColor.b); Serial.println(")");
    lastPrint = millis();
  }
  
  delay(30); // ~33Hz loop
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer**, and **NeoPixel Strip** onto the canvas.
2. Wire Potentiometer to **GPIO34** and NeoPixel DIN to **GPIO13**.
3. Paste the code and click **Run**.
4. Slide the potentiometer widget. Watch the color hue on the NeoPixel strip change dynamically.

## Expected Output
Serial Monitor:
```
RMT NeoPixel driver online.
Pot: 2048 | Hue: 0.50 | RGB: (0,80,80)
Pot: 4095 | Hue: 1.00 | RGB: (80,0,0)
```

NeoPixel Strip Output:
* The entire 16-pixel strip lights up with a solid color that changes smoothly through red, yellow, green, cyan, blue, magenta, and back to red as you adjust the potentiometer.

## Expected Canvas Behavior
* At boot, the NeoPixel strip is OFF.
* Adjusting the potentiometer slider updates the colors of all 16 LEDs.
* The colors change smoothly and dynamically.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `bit0` and `bit1` | Defines the high/low tick configurations for the WS2812B bits (25 ns per tick). |
| `rmt_config(...)` | Configures the RMT transmitter channel, dividing the clock to 40 MHz. |
| `rmt_write_items(...)` | Pushes the encoded color buffer to the transmitter hardware. |

## Hardware & Safety Concept: WS2812B High-Speed Timings and the RMT
WS2812B addressable LEDs use a single-wire protocol with strict timing limits (e.g. bits are only 1.25 microseconds wide, and tolerances are under 150 ns). Standard microcontrollers must disable interrupts and bit-bang the pins, which blocks the CPU from running other code. The ESP32's hardware **RMT (Remote Control) peripheral** acts as a hardware pattern generator: it reads timing structures from RAM and toggles the pin in the background, freeing the CPU to run other code.

## Try This! (Challenges)
1. **Rainbow Cycle Wave visualizer**: Modify the code to generate a scrolling rainbow wave across the 16 pixels.
2. **Interactive speed control**: Connect a button on GPIO 4 that cycles through different animation speeds.
3. **Brightness safety dimmer**: Add a second potentiometer to dynamically adjust the strip's brightness.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| NeoPixels do not light up or flash randomly | Wrong clock division or pin | Verify that the NeoPixel data pin is connected to GPIO 13 and `clk_div` is set to 2 |
| The ESP32 crashes on transmit | Buffer index out of bounds | Check that `TOTAL_ITEMS` is configured to match the number of LEDs (16) |
| Colors are inverted | Color format mismatched | WS2812B uses GRB format; verify the bit shifting sequence in `updateStripColor()` |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [170 - ESP32 NeoPixel Color Spectrum Visualizer](170-esp32-neopixel-color-spectrum-visualizer.md)
- [64 - ESP32 WS2812B NeoPixel Ring Basics](../intermediate/64-esp32-ws2812b-neopixel-ring-basics.md)
- [189 - ESP32 RMT Hardware Infrared Decoder](189-esp32-rmt-hardware-infrared-decoder.md)
