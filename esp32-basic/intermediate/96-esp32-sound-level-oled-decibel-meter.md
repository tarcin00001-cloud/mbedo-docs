# 96 - ESP32 Sound Level OLED Decibel Meter

Build a visual sound level meter that reads analog audio waveforms from a microphone sensor and displays decibel-equivalent values alongside an animated graphical bar on an SSD1306 OLED screen.

## Goal
Learn how to sample high-frequency AC signals from a microphone sensor, calculate peak-to-peak values, map them to a decibel approximation scale, and render them on an OLED.

## What You Will Build
A sound sensor's analog output is connected to GPIO 34. The ESP32 rapidly samples the audio waveform over a 50 ms window, computes the peak-to-peak amplitude, and maps this to a relative decibel scale (dB SPL proxy). The SSD1306 OLED displays the numeric level and a horizontal decibel bar.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| KY-038 Sound Sensor Module | `sound_sensor` | Yes | Yes |
| 0.96″ SSD1306 OLED (I2C) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Sound Sensor | AO | GPIO34 | Yellow | Analog waveform input |
| Sound Sensor | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| SSD1306 OLED | VCC | 3V3 | Red | OLED power |
| SSD1306 OLED | GND | GND | Black | Ground reference |
| SSD1306 OLED | SDA | GPIO21 | Blue | I2C Data line |
| SSD1306 OLED | SCL | GPIO22 | Yellow | I2C Clock line |

> **Wiring tip:** Connect the sound sensor's **AO (analog output)** pin, not the DO pin. The AO pin outputs the raw audio signal oscillating around 1.65V, which must be sampled continuously to find the peak amplitude.

## Code
```cpp
// Sound Level OLED Decibel Meter
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

const int SOUND_PIN = 34;
const int SAMPLE_WINDOW_MS = 50; // 50ms window = ~20Hz updates

void setup() {
  Serial.begin(115200);
  
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("SSD1306 allocation failed!");
    while(true);
  }
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.print("Audio Meter Setup");
  display.display();
  
  delay(1000);
}

void loop() {
  unsigned long start = millis();
  int maxVal = 0;
  int minVal = 4095;
  
  // Sample audio waveform over the window interval
  while (millis() - start < SAMPLE_WINDOW_MS) {
    int sample = analogRead(SOUND_PIN);
    if (sample < 4096) { // Filter out random spikes
      if (sample > maxVal) maxVal = sample;
      if (sample < minVal) minVal = sample;
    }
  }
  
  // Peak-to-peak amplitude
  int peakToPeak = maxVal - minVal;
  
  // Map peak-to-peak amplitude to an approximate decibel scale (dB)
  // Silence ≈ 40dB, Loudest sound ≈ 100dB
  float dbVal = map(peakToPeak, 0, 1500, 40, 100);
  dbVal = constrain(dbVal, 40.0, 100.0);
  
  // Update OLED display
  display.clearDisplay();
  
  // Print Header
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.print("Sound Level Meter");
  
  // Print Decibel Reading
  display.setTextSize(2);
  display.setCursor(0, 20);
  display.print(dbVal, 1);
  display.print(" dB");
  
  // Draw horizontal decibel bar (y = 48 to 58)
  display.drawRect(0, 48, 128, 12, SSD1306_WHITE);
  int barWidth = map((int)dbVal, 40, 100, 0, 124);
  display.fillRect(2, 50, barWidth, 8, SSD1306_WHITE);
  
  display.display();
  
  Serial.print("Peak: "); Serial.print(peakToPeak);
  Serial.print(" | dB: "); Serial.println(dbVal, 1);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Sound Sensor**, and **SSD1306 OLED** onto the canvas.
2. Wire the AO pin to **GPIO34** and the OLED SDA/SCL to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the sound sensor input slider up and down, and observe the decibel values and bar graph change on the OLED.

## Expected Output
Serial Monitor:
```
Peak: 45 | dB: 41.8
Peak: 850 | dB: 74.0
Peak: 1400 | dB: 96.0
```

OLED Layout:
```
Sound Level Meter
74.0 dB
┌────────────────┐
│████████        │  ← Horizontal bar representing relative loudness
└────────────────┘
```

## Expected Canvas Behavior
* The OLED updates its layout at a 20Hz refresh frequency, matching the 50 ms audio sampling window.
* Increasing the sound slider widget on the canvas raises the decibel reading and extends the bar graph dynamically.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `maxVal - minVal` | Calculates peak-to-peak amplitude to measure volume height instead of waveform phase. |
| `map(..., 0, 1500, 40, 100)` | Maps the ADC swing range to human-relatable decibel bounds. |
| `display.fillRect(2, 50, barWidth, ...)` | Draws the filled bar representing current noise volume levels. |

## Hardware & Safety Concept: Sound Pressure Levels (SPL)
Microphone outputs fluctuate rapidly as sound waves compress and decompress the air. A single analog sample can easily read 0 (no pressure) or 4095 (maximum pressure) even during a continuous tone. To measure loudness, we must capture a sequence of samples spanning at least one full cycle of the lowest audio frequency we care about (~20 Hz has a 50 ms period) and measure the difference between the absolute highest and lowest peaks.

## Try This! (Challenges)
1. **Dynamic Peak Hold**: Draw a single vertical line on the bar graph representing the maximum decibel reading recorded in the last 3 seconds.
2. **Noise Alert Threshold**: Flash the OLED screen inversion if the decibel reading exceeds 85 dB (hearing damage warning).
3. **A-Weighting Filter**: Implement basic software smoothing algorithms to prioritize human ear conversation frequencies.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Decibel reading stays fixed at 40 dB | AO pin not connected | Verify the yellow wire connects to the sensor's AO (analog output) terminal, not DO |
| Decibel readings jump erratically in a quiet room | High sensor amplifier gain | Adjust the brass screw potentiometer on the sound sensor to reduce gain |
| Graph is laggy | Screen buffer updates are too slow | Increase the I2C speed using `Wire.setClock(400000)` in `setup()` |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [50 - ESP32 Sound Sensor Threshold Alarm](../beginner/50-esp32-sound-sensor-threshold-alarm.md)
- [42 - ESP32 Sound Sensor Clap Detector](../beginner/42-esp32-sound-sensor-clap-detector.md)
- [60 - ESP32 OLED SSD1306 Display Setup](60-esp32-oled-ssd1306-display-setup.md)
