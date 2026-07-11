# 170 - ESP32 NeoPixel Color Spectrum Visualizer

Build a high-performance audio spectrum visualizer that samples analog audio waveforms from a microphone, calculates frequency components using a Fast Fourier Transform (FFT), and maps the spectrum bands to a 16-pixel WS2812B NeoPixel strip using color-coded frequency zones.

## Goal
Learn how to apply Fast Fourier Transforms (FFT), translate frequency bins into RGB color coordinates, and drive WS2812B NeoPixel digital addressable LED strips.

## What You Will Build
An analog microphone is connected to GPIO 34. A WS2812B NeoPixel strip (16 pixels) is connected to GPIO 13. The ESP32 samples audio waveforms at 9 kHz, processes them using the `arduinoFFT` library, and maps the frequency spectrum onto the 16-pixel strip: Bass (lows) turns the left pixels Red, Mids turn the center pixels Green, and Treble (highs) turns the right pixels Blue.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Microphone Module (Analog Output) | `sound_sensor` | Yes | Yes |
| WS2812B NeoPixel Strip (16 LEDs) | `neopixel` | Yes | Yes |
| External 5V Power Supply | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Microphone Module | AO (Analog) | GPIO34 | Yellow | Audio wave input |
| Microphone Module | VCC / GND | 3V3 / GND | Red / Black | Sensor power |
| NeoPixel Strip | DIN (Data In) | GPIO13 | Green | High-speed data line |
| NeoPixel Strip | 5V / GND | 5V / GND | Red / Black | Shared power rails |

> **Wiring tip:** WS2812B NeoPixels can draw up to 60 mA per pixel at full white brightness. For 16 pixels (max 960 mA), power the strip from the 5V Vin pin (when USB powered) or use an external 5V power supply, sharing the ground with the ESP32.

## Code
```cpp
// NeoPixel Color Spectrum Visualizer (FFT -> NeoPixel Strip)
#include <Adafruit_NeoPixel.h>
#include "arduinoFFT.h"

const int MIC_PIN = 34;
const int NEOPIXEL_PIN = 13;
const int NUM_LEDS = 16;

Adafruit_NeoPixel strip(NUM_LEDS, NEOPIXEL_PIN, NEO_GRB + NEO_KHZ800);

// FFT Parameters
const uint16_t SAMPLES = 128; // Must be power of 2
const double SAMPLING_FREQUENCY = 9000; // Captures audio up to 4.5 kHz

double vReal[SAMPLES];
double vImag[SAMPLES];

arduinoFFT FFT = arduinoFFT(vReal, vImag, SAMPLES, SAMPLING_FREQUENCY);

unsigned int sampling_period_us;
unsigned long microseconds;

void setup() {
  Serial.begin(115200);
  
  sampling_period_us = round(1000000 * (1.0 / SAMPLING_FREQUENCY));
  
  strip.begin();
  strip.show(); // Initialize all pixels to OFF
  strip.setBrightness(50); // Set moderate brightness (safety limit)
  
  Serial.println("NeoPixel Spectrum Visualizer online.");
}

void loop() {
  // 1. High-speed Time-Domain Sampling
  for (int i = 0; i < SAMPLES; i++) {
    microseconds = micros();
    vReal[i] = analogRead(MIC_PIN); 
    vImag[i] = 0;
    
    while (micros() - microseconds < sampling_period_us) {
      // Loop wait
    }
  }
  
  // 2. Perform Fast Fourier Transform (FFT)
  FFT.DCRemoval();
  FFT.Windowing(FFT_WIN_TYP_HAMMING, FFT_FORWARD);
  FFT.Compute(FFT_FORWARD);
  FFT.ComplexToMagnitude();
  
  // 3. Map Frequency Spectrum to 16 NeoPixels
  // We divide the 16 LEDs into three zones:
  // - Pixels 0-4: Bass (Low frequencies, Red)
  // - Pixels 5-10: Mids (Medium frequencies, Green)
  // - Pixels 11-15: Treble (High frequencies, Blue)
  
  // Calculate average magnitudes for each zone
  double bassSum = 0;
  for (int i = 2; i < 7; i++) bassSum += vReal[i]; // Skip DC in bin 0,1
  double bassAvg = bassSum / 5;
  
  double midSum = 0;
  for (int i = 7; i < 20; i++) midSum += vReal[i];
  double midAvg = midSum / 13;
  
  double trebleSum = 0;
  for (int i = 20; i < 50; i++) trebleSum += vReal[i];
  double trebleAvg = trebleSum / 30;
  
  // Map average magnitudes to brightness levels (0-255)
  // Logarithmic mapping for better visual dynamics
  int bassVal   = (int)(log10(bassAvg + 1.0) * 80.0);
  int midVal    = (int)(log10(midAvg + 1.0) * 80.0);
  int trebleVal = (int)(log10(trebleAvg + 1.0) * 80.0);
  
  bassVal   = constrain(bassVal, 0, 255);
  midVal    = constrain(midVal, 0, 255);
  trebleVal = constrain(trebleVal, 0, 255);
  
  // 4. Update NeoPixel Colors
  // Bass Zone (Pixels 0-4): Red
  for (int i = 0; i < 5; i++) {
    strip.setPixelColor(i, strip.Color(bassVal, 0, 0));
  }
  
  // Mid Zone (Pixels 5-10): Green
  for (int i = 5; i < 11; i++) {
    strip.setPixelColor(i, strip.Color(0, midVal, 0));
  }
  
  // Treble Zone (Pixels 11-15): Blue
  for (int i = 11; i < NUM_LEDS; i++) {
    strip.setPixelColor(i, strip.Color(0, 0, trebleVal));
  }
  
  strip.show(); // Render pixel changes
  
  delay(10); // Loop pacing
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Sound Sensor**, and **NeoPixel Strip** onto the canvas.
2. Wire Sound AO to **GPIO34** and NeoPixel DI to **GPIO13**.
3. Paste the code and click **Run**.
4. Slide the frequency and amplitude widgets on the sound sensor. Watch the Red, Green, and Blue zones on the NeoPixel strip illuminate.

## Expected Output
Serial Monitor:
* Real-time audio spectrum color shifts on the NeoPixel strip.

NeoPixel Strip Output:
```
[LED 0-4]   : RED   (Illuminates on Bass beats)
[LED 5-10]  : GREEN (Illuminates on Mid voices)
[LED 11-15] : BLUE  (Illuminates on Treble cymbals)
```

## Expected Canvas Behavior
* At boot, the NeoPixel strip is OFF.
* Adjusting the simulated sound frequency slider shifts the lighting intensity across the color zones:
  - Low frequencies (e.g. 100Hz) illuminate the Red LEDs (0–4).
  - Mid frequencies (e.g. 1kHz) illuminate the Green LEDs (5–10).
  - High frequencies (e.g. 3kHz) illuminate the Blue LEDs (11–15).
* Adjusting the amplitude slider changes the brightness of the active colors.

## Code Walkthrough
| Line | Math / Step |
| --- | --- |
| `strip.begin()` | Initializes the NeoPixel driver library. |
| `bassSum / 5` | Calculates the average magnitude of the low-frequency bins. |
| `strip.Color(bassVal, 0, 0)` | Formats the RGB color value using the scaled bass intensity. |

## Hardware & Safety Concept: NeoPixel Power Demands
WS2812B addressable LEDs contain a small integrated microcontroller inside each LED package. Each of the three color channels (Red, Green, Blue) draws up to 20 mA at full brightness. If all 16 pixels are set to full white brightness (16 x 60 mA = 960 mA), the current exceeds what the ESP32's onboard 3.3V regulator can supply, causing the microcontroller to brown out and reset. Always:
1. Power NeoPixel strips from the 5V Vin line or an external supply.
2. Set software brightness safety limits using `strip.setBrightness(50)`.

## Try This! (Challenges)
1. **Scrolling Rainbow visualizer**: Shift the frequency peaks as a traveling wave across the strip.
2. **Beat detection flash**: flash all 16 LEDs white if the total energy (sum of all bins) spikes suddenly.
3. **Decay Peak indicator**: Implement a peak decay filter so the LED brightness fades out smoothly when sound stops.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| NeoPixels do not light up or flash random colors | Ground reference missing or data pin wrong | Ensure the NeoPixel ground is tied directly to the ESP32 ground; verify data pin is wired to GPIO 13 |
| Red LEDs are bright, but Blue/Green are dim or off | Supply voltage too low | WS2812B requires at least 4.5V to illuminate blue and green colors. Ensure the strip is powered by 5V |
| First pixel is dead | Data line spike | Place a 330 Ω resistor between the ESP32 GPIO 13 and the NeoPixel DIN pin to protect it from voltage spikes |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [156 - ESP32 Sound Level Spectrum Analyzer](156-esp32-sound-level-spectrum-analyzer.md)
- [42 - ESP32 Sound Sensor Clap Detector](../beginner/42-esp32-sound-sensor-clap-detector.md)
- [64 - ESP32 WS2812B NeoPixel Ring Basics](../intermediate/64-esp32-ws2812b-neopixel-ring-basics.md)
