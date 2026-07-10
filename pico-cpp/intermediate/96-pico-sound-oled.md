# 96 - Pico Sound OLED

Build a real-time decibel meter that displays microphone sound levels as a graphic bar on an OLED.

## Goal
Learn how to read analog microphones, calculate average peak amplitudes, and draw graphic bars on SSD1306 screens.

## What You Will Build
A sound volume visualizer:
- **Sound Sensor (GP26)**: Connected to ADC0.
- **SSD1306 OLED (GP4, GP5)**: Displays the sound volume as a horizontal bar graph (from 0 to 128 pixels wide).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Sound Sensor Module | `potentiometer` | Yes (represented by potentiometer slider) | Yes (or microphone module) |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Sound Sensor | VCC | 3.3V | Power supply |
| Sound Sensor | OUT (Analog) | GP26 (ADC0) | Analog volume level |
| Sound Sensor | GND | GND | Ground return |
| SSD1306 OLED | VCC | 3.3V | Display power |
| SSD1306 OLED | SDA | GP4 | I2C Data |
| SSD1306 OLED | SCL | GP5 | I2C Clock |
| SSD1306 OLED | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int MIC_PIN = 26;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(MIC_PIN, INPUT);

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }
}

void loop() {
  int rawVolume = analogRead(MIC_PIN);

  // Map 12-bit input (0-4095) to screen width (0-128 pixels)
  int barWidth = rawVolume * 128 / 4095;

  display.clearDisplay();

  // Print text label
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(10, 10);
  display.print("VOLUME METER");

  // Draw volume bar background boundary
  display.drawRect(0, 30, 128, 20, SSD1306_WHITE);

  // Fill volume bar proportionally
  if (barWidth > 0) {
    display.fillRect(2, 32, barWidth - 4, 16, SSD1306_WHITE);
  }

  display.display();

  delay(50); // Fast update rate for sound spikes
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Sound Sensor** (represented by potentiometer), and **SSD1306 OLED** onto the canvas.
2. Connect Sound Sensor to **GP26**, OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the potentiometer slider representing volume and watch the OLED bar grow.

## Expected Output

Terminal:
```
Simulation active. OLED volume bar updating.
```

## Expected Canvas Behavior
| Slide Position (Volume) | GP26 Read Value | OLED Bar Width |
| --- | --- | --- |
| Leftmost (Quiet) | < 300 | Very narrow / empty |
| Center (Medium) | 2000 | Half filled |
| Rightmost (Loud) | > 3800 | **Fully filled** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `display.fillRect(2, 32, barWidth - 4, 16, ...)` | Draws a solid white block representing the sound volume within the rectangular outline. |

## Hardware & Safety Concept: Peak-to-Peak Amplitude Sampling
Microphone sensor modules output alternating current (AC) wave signals centered around half of VCC (1.65V). Reading a single analog sample can capture the signal at its zero point even during loud sounds. To measure volume accurately on real hardware, software should sample the microphone rapidly for a 50 ms window, calculate the difference between the highest peak and lowest trough (peak-to-peak amplitude), and map that difference to the display.

## Try This! (Challenges)
1. **Critical Overload Alert**: Draw a flashing "OVERLOAD" message if the bar width exceeds 115 pixels.
2. **Dynamic Tone**: Connect a buzzer on GP14 and beep when the volume reaches maximum.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Volume bar flickers constantly | Screen clear rate too high | A 50 ms update rate is normal for sound meters. If flickering is distracting, increase the delay to 100 ms. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [42 - Pico Sound Sensor](../../beginner/42-pico-sound-sensor.md)
- [60 - Pico OLED Setup](60-pico-oled-setup.md)
