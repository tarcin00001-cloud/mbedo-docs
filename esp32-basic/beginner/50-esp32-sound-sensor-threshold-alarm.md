# 50 - ESP32 Sound Sensor Threshold Alarm

Use the analog output of a microphone sound sensor to measure sound levels and trigger a buzzer alarm when the ambient noise exceeds a set threshold.

## Goal
Learn how to read the analog output of a microphone module, compute a peak level from multiple samples, and drive an alarm when environmental noise exceeds a configurable decibel-equivalent threshold.

## What You Will Build
A KY-038 sound sensor module analog output on GPIO 34. The code takes 30 rapid ADC samples, finds the peak value (amplitude of the sound wave), and sounds a buzzer on GPIO 15 if the peak exceeds the threshold.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| KY-038 Sound Sensor Module | `sound_sensor` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| LED (red) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| KY-038 Sound Sensor | VCC | 3V3 | Red | Module supply voltage |
| KY-038 Sound Sensor | GND | GND | Black | Module ground |
| KY-038 Sound Sensor | AO (analog out) | GPIO34 | Yellow | Analog microphone waveform output |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Alarm output |
| Active Buzzer | GND (−) | GND | Black | Ground return |
| LED (red) | Anode (+) | GPIO5 via 330 Ω | Orange | Level threshold indicator |
| LED (red) | Cathode (−) | GND | Black | Ground return |

> **Wiring tip:** Connect the module's **AO** (analog out) pin to GPIO 34 — not DO. The AO pin outputs the raw microphone waveform centred around ~2048 ADC counts (half of 3V3). Measuring the **peak-to-peak amplitude** of multiple fast samples gives a proxy for sound pressure level. The KY-038's sensitivity trimmer adjusts the onboard amplifier gain; higher gain means higher AO swing for the same sound level.

## Code
```cpp
// Sound Sensor Threshold Alarm — analog peak detector
const int  SOUND_PIN    = 34;   // AO — analog waveform
const int  LED_PIN      = 5;    // Alarm LED
const int  BUZZER_PIN   = 15;   // Alarm buzzer

// Sample parameters
const int  SAMPLES      = 30;   // Samples per measurement window
const int  SAMPLE_MS    = 50;   // Total window duration in ms (~1.67 ms per sample)

// Alarm threshold — peak ADC amplitude above baseline
const int  THRESHOLD    = 300;  // Adjust based on environment

int measurePeak() {
  int minVal =  4096;
  int maxVal = -1;

  unsigned long start = millis();
  while (millis() - start < SAMPLE_MS) {
    int sample = analogRead(SOUND_PIN);
    if (sample < minVal) minVal = sample;
    if (sample > maxVal) maxVal = sample;
  }
  return maxVal - minVal;   // Peak-to-peak amplitude
}

void setup() {
  pinMode(LED_PIN,    OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(LED_PIN,    LOW);
  digitalWrite(BUZZER_PIN, LOW);

  Serial.begin(115200);
  Serial.println("Sound Threshold Alarm ready.");
  Serial.print("Alarm threshold (peak ADC): "); Serial.println(THRESHOLD);
}

void loop() {
  int peak = measurePeak();

  Serial.print("Peak amplitude: "); Serial.print(peak);

  if (peak > THRESHOLD) {
    digitalWrite(LED_PIN,    HIGH);
    digitalWrite(BUZZER_PIN, HIGH);
    Serial.println("  !! NOISE ALARM !!");
    delay(500);   // Hold alarm for 500 ms per trigger
    digitalWrite(LED_PIN,    LOW);
    digitalWrite(BUZZER_PIN, LOW);
  } else {
    Serial.println("  — Quiet");
    delay(100);
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Sound Sensor**, **Buzzer**, and **LED** onto the canvas.
2. Connect Sound Sensor **AO** to **GPIO34**, Buzzer to **GPIO15**, LED to **GPIO5**.
3. Paste the code and click **Run**.
4. Increase the sound sensor slider in the canvas to simulate a loud environment — alarm fires.

## Expected Output
Serial Monitor:
```
Sound Threshold Alarm ready.
Alarm threshold (peak ADC): 300
Peak amplitude: 45   — Quiet
Peak amplitude: 88   — Quiet
Peak amplitude: 520  !! NOISE ALARM !!
Peak amplitude: 312  !! NOISE ALARM !!
Peak amplitude: 62   — Quiet
```

## Expected Canvas Behavior
* LED and buzzer are inactive during quiet conditions (peak below threshold).
* Moving the sound sensor slider above the threshold level activates the LED and buzzer for 500 ms.
* After 500 ms the alarm resets and re-evaluates on the next measurement cycle.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `while (millis() - start < SAMPLE_MS)` | Samples the ADC as fast as possible for 50 ms — captures multiple waveform cycles. |
| `if (sample < minVal) minVal = sample` | Tracks the minimum ADC value in the window (negative peak). |
| `if (sample > maxVal) maxVal = sample` | Tracks the maximum ADC value in the window (positive peak). |
| `return maxVal - minVal` | Peak-to-peak amplitude: a proxy for sound pressure. Quiet ~0–50, whisper ~50–150, loud clap ~400–800. |
| `if (peak > THRESHOLD)` | Fires the alarm when the amplitude exceeds the configured noise floor. |
| `delay(500)` | Holds the alarm on for 500 ms to make it noticeable before re-evaluating. |

## Hardware & Safety Concept: Peak-to-Peak Amplitude as a Sound Level Proxy
A microphone converts sound pressure waves into a tiny AC electrical signal centred on a DC bias voltage. In silence, the ADC reads approximately 2048 (half of 3.3 V). A sound wave causes the voltage to oscillate above and below this centre — louder sounds produce larger oscillations. The **peak-to-peak amplitude** (maximum − minimum ADC value over a short window) measures how large these oscillations are, which is directly related to sound pressure level. Doubling sound pressure (≈6 dB SPL increase) approximately doubles the peak-to-peak amplitude. This approach is used in noise-monitoring devices, smart office pods (alert when noise exceeds conversation levels), airport runway noise monitors, and baby cry detectors.

## Try This! (Challenges)
1. **Adjustable threshold**: Read a potentiometer on GPIO 35 and map its value to set the alarm threshold dynamically.
2. **Noise level bar**: Map the peak amplitude to 0–5 and light that many LEDs as a visual noise bar graph.
3. **Calibration mode**: Print peak amplitude readings for 30 seconds in a quiet environment to find the natural noise floor, then set `THRESHOLD` to that value + 100.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm always firing in quiet room | Threshold too low | Increase `THRESHOLD` to 500 or higher |
| Alarm never fires even with loud sounds | Sensitivity trimmer at minimum | Rotate the KY-038 trimmer clockwise; shout directly at the microphone to confirm AO changes |
| Peak amplitude always reads 0 | AO pin not connected or wired to DO pin | Verify the yellow wire connects to the KY-038's AO pin, not DO |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [42 - ESP32 Sound Sensor Clap Detector](42-esp32-sound-sensor-clap-detector.md)
- [38 - ESP32 Piezo Vibration Analog Signal Reader](38-esp32-piezo-vibration-analog-signal-reader.md)
- [43 - ESP32 MQ-2 Gas Sensor Level Print](43-esp32-mq2-gas-sensor-level-print.md)
