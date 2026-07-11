# 156 - Sound Level Spectrum Analyzer

Analyze audio signals from an analog sound sensor connected to the ADC0 (GP26) input, filtering the signal into frequency bands and visualizing the spectrum on an SSD1306 OLED display.

## Goal
Learn how to design and execute digital Infinite Impulse Response (IIR) frequency filters on the VEGA ARIES v3 board without C++ loops or array buffers to decompose audio into spectrum bands.

## What You Will Build
An audio spectrum visualizer. An analog sound sensor (microphone breakout) captures audio waveforms on pin ADC0. The software processes the input stream using low-pass, band-pass, and high-pass digital equations to estimate the intensity of Bass, Mid, Treble, and Presence frequencies. These values are drawn as dynamic bar graphs on an SSD1306 OLED screen.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Analog Sound Sensor | `sound_sensor` | Yes | Yes |
| 0.96" SSD1306 OLED Display | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Sound Sensor | VCC | 3V3 | Red | Power supply (3.3V) |
| Sound Sensor | GND | GND | Black | Ground reference |
| Sound Sensor | OUT (Analog) | ADC0 (GP26) | Blue | Analog output signal |
| SSD1306 OLED | VCC | 3V3 | Red | Power supply (3.3V) |
| SSD1306 OLED | GND | GND | Black | Ground reference |
| SSD1306 OLED | SDA | SDA0 (GP17) | Blue | Shared I2C Data line |
| SSD1306 OLED | SCL | SCL0 (GP16) | Yellow | Shared I2C Clock line |

> **Wiring tip:** Standard microphone sensors have a potentiometer to adjust gain. Calibrate it so that silent rooms read around the ADC midpoint (512 out of 1023).

## Code
```cpp
// Sound Level Spectrum Analyzer - VEGA ARIES v3
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

const int SOUND_PIN = GP26; // Analog Sound Sensor on ADC0 (GP26)

// Frequency Band States (Single-variable IIR Filters)
float bass = 0.0;
float mid = 0.0;
float treble = 0.0;
float presence = 0.0;

unsigned long lastTime = 0;

void setup() {
  Serial.begin(115200);
  Wire.begin();

  display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR);
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.print("Audio Spectrometer");
  display.display();

  pinMode(SOUND_PIN, INPUT);
  lastTime = millis();
}

void loop() {
  int rawVal = analogRead(SOUND_PIN);

  // Center the sound signal around virtual zero (512 midpoint)
  float audioSample = (float)(rawVal - 512);
  
  // Obtain absolute amplitude value
  if (audioSample < 0.0) {
    audioSample = -audioSample;
  }

  // --- DIGITAL IIR FILTERS (NO ARRAYS OR LOOPS) ---
  // Bass: Low-pass filter (integrates slowly)
  bass = (0.94 * bass) + (0.06 * audioSample);

  // Mid: Band-pass filter (removes bass offset and rejects high treble)
  float bandPassMid = audioSample - bass;
  if (bandPassMid < 0.0) {
    bandPassMid = 0.0;
  }
  mid = (0.82 * mid) + (0.18 * bandPassMid);

  // Treble: High-pass filter (high frequency response)
  float highPassTreble = audioSample - bass - mid;
  if (highPassTreble < 0.0) {
    highPassTreble = 0.0;
  }
  treble = (0.45 * treble) + (0.55 * highPassTreble);

  // Presence: Rapid peaks
  presence = (0.15 * presence) + (0.85 * audioSample);

  unsigned long currentTime = millis();
  unsigned long elapsed = currentTime - lastTime;

  // Refresh visual display at 20Hz (every 50ms)
  if (elapsed >= 50) {
    lastTime = currentTime;

    // Convert frequency energy levels to screen height pixels (0 to 48 max height)
    int h1 = (int)(bass * 0.18);
    int h2 = (int)(mid * 0.25);
    int h3 = (int)(treble * 0.32);
    int h4 = (int)(presence * 0.12);

    // Bound values to prevent screen drawing issues
    if (h1 > 48) h1 = 48;
    if (h2 > 48) h2 = 48;
    if (h3 > 48) h3 = 48;
    if (h4 > 48) h4 = 48;

    display.clearDisplay();
    display.setTextSize(1);
    display.setCursor(0, 0);
    display.print("AUDIO SPECTRUM");

    // Draw vertical spectrum bars
    // Bass Bar
    display.fillRect(12, 60 - h1, 16, h1, SSD1306_WHITE);
    // Mid Bar
    display.fillRect(40, 60 - h2, 16, h2, SSD1306_WHITE);
    // Treble Bar
    display.fillRect(68, 60 - h3, 16, h3, SSD1306_WHITE);
    // Presence Bar
    display.fillRect(96, 60 - h4, 16, h4, SSD1306_WHITE);

    // Draw labels at the base
    display.setCursor(12, 60);
    display.print("B");
    display.setCursor(40, 60);
    display.print("M");
    display.setCursor(68, 60);
    display.print("T");
    display.setCursor(96, 60);
    display.print("P");

    display.display();

    // Export raw values to plot
    Serial.print("Bass:");
    Serial.print(h1);
    Serial.print(",Mid:");
    Serial.print(h2);
    Serial.print(",Treble:");
    Serial.println(h3);
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, **Analog Sound Sensor**, and **SSD1306 OLED** onto the canvas.
2. Wire the Sound Sensor: **VCC** to **3V3**, **GND** to **GND**, and **OUT** to **ADC0 (GP26)**.
3. Wire the OLED: **VCC** to **3V3**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
4. Copy the project code and paste it into the editor.
5. Click **Run**.
6. Alter the microphone volume level slider on the canvas and observe the live changes in the spectrum bars on the OLED.

## Expected Output
Serial Monitor:
```
Bass:4,Mid:12,Treble:15
Bass:18,Mid:22,Treble:8
Bass:35,Mid:10,Treble:3
```

## Expected Canvas Behavior
* The screen displays the four bar charts with labels `B`, `M`, `T`, and `P`.
* When low-frequency inputs occur (bass hums), the `B` bar rises high.
* High-frequency sounds (snaps, whistles) cause the `T` and `P` bars to spike up rapidly.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `rawVal - 512` | Centers the 0-1023 input signal around zero to calculate alternating current (AC) amplitude. |
| `bass = (0.94 * bass) + ...` | Implements a digital low-pass filter to isolate slow-moving volume trends (Bass). |
| `audioSample - bass` | Removes low-frequency components from the sample to isolate mids and highs. |
| `display.fillRect(...)` | Draws solid rectangular bars on the coordinate grid based on calculated heights. |
| `display.display()` | Commits the updated visual buffer to the OLED screen. |

## Hardware & Safety Concept
* **DC Bias Removal**: Microphone outputs contain a DC offset voltage (typically 1.65V) to allow the AC audio signal to swing positive and negative. Subtracting 512 translates this offset to 0, which is necessary to correctly determine filter amplitudes.
* **Sampling Rate Stability**: Sound processing requires uniform sampling. Keeping the main loop free of heavy calculations and blocking delays ensures accurate frequency estimation.

## Try This! (Challenges)
1. **LED Beat Indicator**: Flash the onboard Red LED (`LED_R` / GPIO 23) whenever the Bass bar height `h1` exceeds 35 pixels, indicating a loud beat.
2. **Frequency Peak Hold**: Track the highest level reached for each bar and draw a single dot above the bar representing the peak level, decaying it slowly over time.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| All bars stay flat at the bottom | Microphone offset is incorrect | Verify `rawVal` averages around 512 under silent conditions; if it is 0, the microphone lacks power or is connected to the wrong pin. |
| OLED screen updates are extremely laggy | Loop contains heavy blocking functions | Ensure no `delay()` commands exist in the primary loop besides the 50ms display throttling check. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [30 - Sound Sensor Clapper](../beginner/30-sound-sensor-clapper.md)
- [96 - Sound Level OLED Decibel Meter](../intermediate/96-sound-level-oled-decibel-meter.md)
