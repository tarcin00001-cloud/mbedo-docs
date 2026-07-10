# 38 - ESP32 Piezo Vibration Analog Signal Reader

Read the analog output of a piezo vibration sensor and print the raw signal amplitude to detect and classify impact events.

## Goal
Learn how a piezoelectric disc generates a brief voltage spike on impact, how to capture it with the ESP32 ADC, and how to threshold the reading to detect and classify knock events.

## What You Will Build
A piezo disc sensor with a bleed resistor connected to GPIO 34. When the disc is tapped or vibrated, the piezo generates a voltage spike. The code reads the ADC, detects peaks above a threshold, and classifies the impact as Light, Medium, or Heavy.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Piezo Disc (bare element) | `piezo` | Yes | Yes |
| 1 MΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Piezo Disc | Positive (red wire / brass side) | GPIO34 | Yellow | Signal output — ADC input |
| Piezo Disc | Negative (black wire / silver side) | GND | Black | Ground reference |
| 1 MΩ Resistor | Leg 1 | GPIO34 | White | Bleed resistor — clamps ringing to 0 V at rest |
| 1 MΩ Resistor | Leg 2 | GND | Black | Ground reference |

> **Wiring tip:** The 1 MΩ bleed resistor is essential — it slowly discharges the piezo's piezoelectric capacitance back to 0 V between impacts. Without it, the ADC reads a slowly drifting charge and misses fast spikes. Do NOT use the piezo buzzer/module for this project — use the bare piezo disc element. The disc also generates negative voltage spikes on release; the ESP32 ADC clamps negative voltages to 0 V internally.

## Code
```cpp
// Piezo Vibration Analog Signal Reader
const int PIEZO_PIN   = 34;
const int LIGHT_THOLD = 100;    // ADC value for light tap
const int HEAVY_THOLD = 800;    // ADC value for heavy knock

void setup() {
  Serial.begin(115200);
  Serial.println("Piezo Vibration Reader ready — tap the disc.");
}

void loop() {
  int raw = analogRead(PIEZO_PIN);

  if (raw >= HEAVY_THOLD) {
    Serial.print("HEAVY IMPACT  | ADC: ");
    Serial.println(raw);
  } else if (raw >= LIGHT_THOLD) {
    Serial.print("Light tap     | ADC: ");
    Serial.println(raw);
  }
  // Below LIGHT_THOLD: no print — reduces noise chatter

  delay(10);   // 100 Hz sampling to catch fast spikes
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Piezo** sensor onto the canvas.
2. Connect Piezo **signal** to **GPIO34**.
3. Paste the code and click **Run**.
4. Click or shake the piezo widget on the canvas to simulate taps.

## Expected Output
Serial Monitor:
```
Piezo Vibration Reader ready — tap the disc.
Light tap     | ADC: 215
Light tap     | ADC: 312
HEAVY IMPACT  | ADC: 1840
Light tap     | ADC: 490
```

## Expected Canvas Behavior
* No output during idle (ADC remains below the light threshold).
* Clicking the piezo widget generates a value spike — classified by intensity.
* Heavy clicks print the HEAVY IMPACT label and higher ADC values.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `const int LIGHT_THOLD = 100` | Minimum ADC spike to qualify as a deliberate tap vs ambient noise. |
| `const int HEAVY_THOLD = 800` | Threshold above which the impact is classified as heavy. |
| `if (raw >= HEAVY_THOLD)` | Checks for heavy impact first (strongest condition). |
| `else if (raw >= LIGHT_THOLD)` | Falls through to classify as a light tap. |
| `delay(10)` | 100 Hz sampling — essential to catch the brief piezo voltage spike (lasts ~1–10 ms). |

## Hardware & Safety Concept: Piezoelectric Effect
A piezoelectric disc is made from a ceramic (PZT) material bonded to a metal disc. When mechanically deformed (bent by an impact), the crystal lattice produces an electric charge — this is the **direct piezoelectric effect**. The brief voltage spike (typically 0.5–5 V peak for moderate taps) is proportional to the rate of deformation. The 1 MΩ bleed resistor forms an RC circuit with the piezo's internal capacitance (~20 nF), creating a time constant of ~20 ms. This means the voltage decays back to 0 V within ~100 ms after each impact. The same disc driven electrically produces sound — the **inverse piezoelectric effect** — which is how buzzer modules work. Piezo transducers are used in knock sensors, microphones, accelerometers, medical ultrasound probes, and industrial flow meters.

## Try This! (Challenges)
1. **LED burst**: Flash an LED for 200 ms on each detected heavy impact.
2. **Knock counter**: Count knocks in a 2-second window — 3 knocks = secret pattern detected.
3. **Calibration mode**: Print every ADC value for 10 seconds to determine your own light/heavy thresholds for your specific disc.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Constant random values printed | Missing 1 MΩ bleed resistor | Add the resistor between GPIO 34 and GND |
| No output even on hard taps | Piezo positive lead not on GPIO 34 | Swap and re-seat the disc leads |
| ADC always reads 4095 | Piezo spike exceeds 3.3 V | Add a 10 kΩ series resistor between piezo and GPIO 34 to attenuate large spikes |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [29 - ESP32 Vibration Sensor Latch Alarm](29-esp32-vibration-sensor-latch-alarm.md)
- [37 - ESP32 IR Obstacle Sensor Print](37-esp32-ir-obstacle-sensor-print.md)
- [42 - ESP32 Sound Sensor Clap Detector](42-esp32-sound-sensor-clap-detector.md)
