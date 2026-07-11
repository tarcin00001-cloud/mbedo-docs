# 186 - ESP32 External I2S DAC Audio Player

Build a digital audio synthesis station on the ESP32 that interfaces with a MAX98357A external I2S Class-D amplifier over the Inter-IC Sound (I2S) digital audio bus, generating high-fidelity sine wave tones and using a potentiometer to dynamically control the pitch.

## Goal
Learn how to configure the ESP32 hardware I2S peripheral, allocate DMA sound buffers, stream 16-bit digital audio data, and interface with external audio DACs.

## What You Will Build
An external MAX98357A I2S DAC module is connected via digital audio lines (LRC: 25, BCLK: 26, DIN: 22). A potentiometer on GPIO 34 controls the generated tone pitch. An 8 Ω speaker is connected to the amplifier. The ESP32 calculates 16-bit signed sine wave samples and streams them in real time over I2S to play a smooth, clean tone.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MAX98357A I2S DAC Module | `i2s_dac` | Yes | Yes |
| 8 Ω Speaker (2W) | `speaker` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MAX98357A | LRC (WS) | GPIO25 | Yellow | I2S Word Select (Left/Right clock) |
| MAX98357A | BCLK | GPIO26 | Orange | I2S Bit Clock |
| MAX98357A | DIN | GPIO22 | Green | I2S Serial Data line |
| MAX98357A | Vin / GND | 5V / GND | Red / Black | Amplifier power rails |
| Speaker | Terminals | Out+ / Out− | Red / Black | Output screw terminals |
| Potentiometer | Wiper | GPIO34 | Yellow | Pitch adjust wiper |
| Potentiometer | VCC / GND | 3V3 / GND | Red / Black | Sensor power rails |

> **Wiring tip:** The MAX98357A is a high-power Class-D amplifier. Power it from the 5V Vin rail to maximize audio output power. Keep the I2S digital lines short to avoid high-frequency signal interference.

## Code
```cpp
// External I2S DAC Audio Player (MAX98357A + Potentiometer)
#include <Arduino.h>
#include "driver/i2s.h"
#include <math.h>

const int POT_PIN = 34;

// I2S Hardware Pin Assignments
#define I2S_LRC_PIN  25 // Word Select (WS)
#define I2S_BCLK_PIN 26 // Bit Clock (BCLK)
#define I2S_DOUT_PIN 22 // Data Out (DIN)

#define I2S_PORT     I2S_NUM_0
#define SAMPLE_RATE  16000 // 16 kHz sampling rate

// Sine wave configuration
float waveAngle = 0.0;
float targetFrequency = 440.0; // Standard A4 note default

void initI2S() {
  // 1. Configure I2S Driver Settings
  i2s_config_t i2s_config = {
    .mode = (i2s_mode_t)(I2S_MODE_MASTER | I2S_MODE_TX), // Master Transmitter
    .sample_rate = SAMPLE_RATE,
    .bits_per_sample = I2S_BITS_PER_SAMPLE_16BIT,       // 16-bit resolution
    .channel_format = I2S_CHANNEL_FMT_ONLY_LEFT,        // Mono audio stream
    .communication_format = I2S_COMM_FORMAT_STAND_I2S,  // Standard I2S protocol
    .intr_alloc_flags = ESP_INTR_FLAG_LEVEL1,           // Default interrupt priority
    .dma_buf_count = 8,                                 // Number of DMA buffers
    .dma_buf_len = 64,                                  // Size of each buffer
    .use_apll = false,                                  // Don't use high-precision audio PLL
    .tx_desc_auto_clear = true                          // Auto-clear TX descriptor on underflow
  };

  // 2. Configure I2S Hardware Pin Bindings
  i2s_pin_config_t pin_config = {
    .bck_io_num = I2S_BCLK_PIN,
    .ws_io_num = I2S_LRC_PIN,
    .data_out_num = I2S_DOUT_PIN,
    .data_in_io_num = I2S_PIN_NO_CHANGE
  };

  // 3. Initialize I2S port
  i2s_driver_install(I2S_PORT, &i2s_config, 0, NULL);
  i2s_set_pin(I2S_PORT, &pin_config);
  i2s_zero_dma_buffer(I2S_PORT); // Clear buffer on startup
}

void setup() {
  Serial.begin(115200);
  pinMode(POT_PIN, INPUT);
  
  initI2S();
  Serial.println("I2S Audio Output Driver active.");
}

void loop() {
  // Read potentiometer and map to tone frequency (200Hz to 1200Hz)
  int potRaw = analogRead(POT_PIN);
  targetFrequency = 200.0 + (potRaw * 1000.0 / 4095.0);
  
  // Calculate wave angle step based on frequency and sample rate
  // Step = 2 * pi * F / Fs
  float angleStep = (2.0 * M_PI * targetFrequency) / (float)SAMPLE_RATE;
  
  // Buffer to store audio samples for this cycle (32 samples)
  const int BUFFER_LEN = 32;
  int16_t sampleBuffer[BUFFER_LEN];
  
  // Fill sample buffer
  for (int i = 0; i < BUFFER_LEN; i++) {
    // Generate 16-bit signed sine wave sample (-32768 to 32767)
    // Scale amplitude to 40% (approx 13000) to protect speaker from clipping
    sampleBuffer[i] = (int16_t)(13000.0 * sin(waveAngle));
    
    waveAngle += angleStep;
    if (waveAngle >= 2.0 * M_PI) {
      waveAngle -= 2.0 * M_PI;
    }
  }
  
  // Stream buffer to the I2S DAC using DMA
  size_t bytesWritten;
  i2s_write(
    I2S_PORT,
    &sampleBuffer,
    sizeof(sampleBuffer),
    &bytesWritten,
    portMAX_DELAY // Block task until transfer completes
  );
  
  // Print active frequency
  static unsigned long lastPrint = 0;
  if (millis() - lastPrint >= 500) {
    Serial.print("Freq: "); Serial.print(targetFrequency, 1);
    Serial.print(" Hz | Bytes Written: "); Serial.println(bytesWritten);
    lastPrint = millis();
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MAX98357A DAC**, and **Potentiometer** onto the canvas.
2. Wire LRC to **GPIO25**, BCLK to **GPIO26**, DIN to **GPIO22**, and Potentiometer to **GPIO34**.
3. Paste the code and click **Run**.
4. Adjust the potentiometer widget. Watch the frequency value update in the Serial Monitor.
5. In hardware, listen to the speaker output pitch sweep in response.

## Expected Output
Serial Monitor:
```
I2S Audio Output Driver active.
Freq: 440.0 Hz | Bytes Written: 64
Freq: 652.4 Hz | Bytes Written: 64
Freq: 880.0 Hz | Bytes Written: 64
```

Speaker Sound Output:
* A continuous, clean sine wave tone sweeps in pitch from 200 Hz to 1200 Hz.

## Expected Canvas Behavior
* At startup, the I2S peripheral configures.
* Adjusting the potentiometer slider updates the output frequency logs on the Serial Monitor.
* The DAC module widget indicates active data transmission.

## Code Walkthrough
| Line | Math / Step |
| --- | --- |
| `I2S_BITS_PER_SAMPLE_16BIT` | Sets the audio resolution to 16-bit CD-quality digital samples. |
| `i2s_driver_install(...)` | Installs the I2S driver and allocates internal DMA audio transfer buffers. |
| `(13000.0 * sin(waveAngle))` | Generates a 16-bit signed audio sample, scaling the amplitude to prevent clipping. |
| `i2s_write(...)` | Writes the sample buffer directly to the I2S DAC using DMA. |

## Hardware & Safety Concept: Digital Audio Buses and the I2S Protocol
The internal DACs of microcontrollers are often limited to 8-bit resolution, which generates a noticeable background hum (quantization noise). Professional audio hardware uses the **I2S (Inter-IC Sound)** serial bus protocol. It splits audio transmissions into three lines:
1. **BCLK (Bit Clock)**: Coordinates data bits.
2. **LRC (Word Select)**: Toggles between Left and Right stereo channels.
3. **DIN (Serial Data)**: Carries the multi-bit digital audio samples.
Using an external I2S DAC (like the MAX98357A) allows 16-bit or 24-bit high-fidelity audio output.

## Try This! (Challenges)
1. **Musical Scale Keyboard**: Wire four buttons (GPIO 4, 12, 13, 14) and program them to play four notes (e.g. C, E, G, C) when pressed.
2. **Audio Waveform Switcher**: Add a button (GPIO 15) to cycle the generated waveform between Sine, Triangle, and Sawtooth.
3. **WAV File Player**: Integrate an SD card reader (Project 136) to read raw WAV audio files and stream them to the speaker.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sound is noisy or static | Buffer underflow | Ensure the loop runs fast enough; do not include slow `delay()` or serial prints in the loop |
| No sound from speaker | Pin mapping conflict | Verify BCLK is on GPIO 26, LRC on GPIO 25, and DIN on GPIO 22 |
| Audio clips or sounds distorted | Amplitude setting too high | Reduce the amplitude scaling factor in the code (e.g. from 13000 to 8000) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [184 - ESP32 DAC Analog Waveform Generator](184-esp32-dac-analog-waveform-generator.md)
- [181 - ESP32 Custom Hardware Timer Interrupts](181-esp32-custom-hardware-timer-interrupts.md)
- [136 - ESP32 SD Card Data Logger](136-esp32-sd-card-data-logger.md)
