# 156 - ESP32 Sound Level Spectrum Analyzer

Build an audio spectrum analyzer that samples analog waveforms from a microphone at high frequency, processes them using a Fast Fourier Transform (FFT) library, and draws a real-time 8-band frequency bar graph on an SSD1306 OLED screen.

## Goal
Learn how to capture audio waveforms using high-speed microsecond analog sampling, apply Fast Fourier Transforms (FFT) to convert time-domain signals to frequency-domain magnitudes, and draw real-time bar graphs.

## What You Will Build
An analog microphone is connected to GPIO 34. An SSD1306 OLED is on I2C (GPIO 21/22). The ESP32 takes 128 samples at a 9 kHz sampling rate, processes them using the `arduinoFFT` library, groups the frequency bins into 8 bands (from bass to treble), and draws 8 vertical bars on the OLED representing the live audio spectrum.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Microphone Module (Analog Output) | `sound_sensor` | Yes | Yes |
| 0.96″ SSD1306 OLED (I2C) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Microphone Module | AO (Analog) | GPIO34 | Yellow | Audio waveform input |
| Microphone Module | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| SSD1306 OLED | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C shared lines |
| SSD1306 OLED | VCC / GND | 3V3 / GND | Red / Black | OLED power |

> **Wiring tip:** Connect the microphone's analog output (AO) pin directly to GPIO 34. Avoid using the digital output (DO) pin for spectrum analysis. Keep power lines clean to avoid hum noise in the analog readings.

## Code
```cpp
// Sound Level Spectrum Analyzer (FFT on Analog Sound)
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include "arduinoFFT.h"

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

const int MIC_PIN = 34;

// FFT Parameters
// Must be power of 2
const uint16_t SAMPLES = 128; 
// Sampling frequency in Hz (captures audio up to 4.5 kHz)
const double SAMPLING_FREQUENCY = 9000; 

double vReal[SAMPLES];
double vImag[SAMPLES];

// Instantiate FFT object
arduinoFFT FFT = arduinoFFT(vReal, vImag, SAMPLES, SAMPLING_FREQUENCY);

// Sampling period in microseconds
unsigned int sampling_period_us;
unsigned long microseconds;

void setup() {
  Serial.begin(115200);
  
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED allocation failed!");
    while(1) {}
  }
  
  sampling_period_us = round(1000000 * (1.0 / SAMPLING_FREQUENCY));
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(15, 20);
  display.print("Spectrum Analyzer");
  display.display();
  delay(1500);
}

void loop() {
  // 1. High-speed Time-Domain Sampling
  for (int i = 0; i < SAMPLES; i++) {
    microseconds = micros();
    
    // Read raw AC-coupled wave centered around ~1.65V
    vReal[i] = analogRead(MIC_PIN); 
    vImag[i] = 0; // Imaginary part is 0
    
    // Wait for the next sampling period
    while (micros() - microseconds < sampling_period_us) {
      // Loop wait
    }
  }
  
  // 2. Perform Fast Fourier Transform (FFT)
  FFT.DCRemoval(); // Subtract DC offset (averages out center bias)
  FFT.Windowing(FFT_WIN_TYP_HAMMING, FFT_FORWARD); // Apply Hamming window to reduce spectral leakage
  FFT.Compute(FFT_FORWARD); // Compute FFT
  FFT.ComplexToMagnitude(); // Compute magnitudes of frequency components
  
  // 3. Render 8-band Frequency Bar Graph on OLED
  display.clearDisplay();
  
  // Display Header
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.print("FFT AUDIO SPECTRUM");
  display.drawFastHLine(0, 10, 128, SSD1306_WHITE);
  
  // The first SAMPLES/2 bins are valid (Nyquist Theorem limit: 4.5kHz)
  // Divide SAMPLES/2 (64 bins) into 8 frequency bands
  const int BANDS = 8;
  const int BINS_PER_BAND = 8;
  const int BAR_WIDTH = 12;
  const int BAR_SPACING = 3;
  const int X_OFFSET = 6;
  const int GRAPH_MAX_HEIGHT = 45;
  const int Y_BOTTOM = 63;
  
  for (int i = 0; i < BANDS; i++) {
    double bandMagnitude = 0;
    
    // Sum magnitudes for the bins in this band (ignoring DC in band 0)
    int startBin = i * BINS_PER_BAND;
    if (i == 0) startBin = 2; // Skip first 2 bins to filter out residual low frequency noise
    
    for (int j = 0; j < BINS_PER_BAND; j++) {
      bandMagnitude += vReal[startBin + j];
    }
    bandMagnitude /= BINS_PER_BAND; // Average magnitude
    
    // Logarithmic scale mapping for better visual response
    int barHeight = (int)(log10(bandMagnitude + 1.0) * 12.0);
    barHeight = constrain(barHeight, 0, GRAPH_MAX_HEIGHT);
    
    // Calculate screen coordinate
    int x = X_OFFSET + i * (BAR_WIDTH + BAR_SPACING);
    int y = Y_BOTTOM - barHeight;
    
    // Draw vertical bar representing this band
    display.fillRect(x, y, BAR_WIDTH, barHeight, SSD1306_WHITE);
  }
  
  display.display();
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Sound Sensor**, and **SSD1306 OLED** onto the canvas.
2. Wire Sound AO to **GPIO34** and OLED to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Adjust the frequency or amplitude sliders on the simulated sound widget. Watch the corresponding frequency bars jump on the OLED screen.

## Expected Output
Serial Monitor:
* Real-time spectrum bar heights update on screen.

OLED Display Layout:
```
FFT AUDIO SPECTRUM
──────────────────
   █   █
   █ █ █   █
 █ █ █ █ █ █ █ █
 8-Band Frequency Bars (Bass → Treble)
```

## Expected Canvas Behavior
* At boot, the OLED displays flat spectrum bars.
* Adjusting the simulated sound frequency slider on the sound widget shifts the height peak across the 8 bars (low frequency slider = left bars rise; high frequency = right bars rise).
* Increasing the amplitude slider increases the heights of the bars.

## Code Walkthrough
| Line | Math / Action |
| --- | --- |
| `FFT.DCRemoval()` | Subtracts the DC bias (approx 1.65V) from the samples to align the audio waveform around 0. |
| `FFT.Compute(...)` | Performs the FFT mathematical calculations, converting the signal to the frequency domain. |
| `log10(bandMagnitude + 1.0) * 12.0` | Maps the raw magnitude values logarithmically to match human hearing perception. |

## Hardware & Safety Concept: The Nyquist Theorem and Spectral Leakage
To digitize an audio signal accurately, you must sample at a rate at least **twice** the highest frequency you want to measure (the **Nyquist Rate**). Sampling at 9 kHz allows capturing audio frequencies up to 4.5 kHz. **Windowing functions** (like the Hamming Window) are applied to the raw samples to taper the ends of the sample buffer to zero. This prevents spectral leakage, which creates false frequency peaks at the edges of the frequency bands.

## Try This! (Challenges)
1. **Dynamic Peaks indicator**: Draw a single floating pixel above each bar that slowly decays downwards, representing peak hold levels.
2. **Frequency Alert trigger**: Sound a buzzer (GPIO 15) if high-frequency treble bands (Band 6 or 7) exceed a target magnitude.
3. **Double FFT resolution**: Increase `SAMPLES` to 256 and sample at 12 kHz to plot a detailed 16-band spectrum.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| All frequency bars are flat | Microphone output pin floating | Ensure the microphone's analog output (AO) pin is connected to GPIO 34, not the digital (DO) pin |
| First bar is always maxed out | Residual DC bias | Adjust the `DCRemoval()` command or skip the first 3 low frequency bins |
| Refresh rate is sluggish | FFT calculation overhead | Keep `SAMPLES` to 128 to ensure the ESP32 can complete the calculations quickly |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [96 - ESP32 Sound Level OLED Decibel Meter](../intermediate/96-esp32-sound-level-oled-decibel-meter.md)
- [42 - ESP32 Sound Sensor Clap Detector](../beginner/42-esp32-sound-sensor-clap-detector.md)
- [170 - ESP32 NeoPixel Color Spectrum Visualizer](170-esp32-neopixel-color-spectrum-visualizer.md)
