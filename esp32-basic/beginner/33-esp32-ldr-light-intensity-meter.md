# 33 - ESP32 LDR Light Intensity Meter

Read ambient light levels with a Light Dependent Resistor (LDR) and print the ADC value and a human-readable light label to the Serial Monitor.

## Goal
Learn how an LDR in a voltage divider circuit converts light intensity into an analog voltage that the ESP32 ADC can measure, and how to categorise the reading into light level bands.

## What You Will Build
An LDR and a 10 kΩ fixed resistor form a voltage divider on GPIO 34. Bright light lowers the LDR resistance, raising the wiper voltage. The code reads the ADC and classifies the level as Dark, Dim, Moderate, Bright, or Very Bright.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LDR (photoresistor) | `ldr` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR | Leg 1 | 3V3 | Red | Supply rail |
| LDR | Leg 2 | GPIO34 (and resistor Leg 1) | Yellow | Voltage divider midpoint — ADC input |
| 10 kΩ Resistor | Leg 1 | GPIO34 | White | Pull-down leg of divider |
| 10 kΩ Resistor | Leg 2 | GND | Black | Ground rail |

> **Wiring tip:** The LDR and resistor form a voltage divider: LDR on top (3V3 side), fixed resistor on the bottom (GND side). The ADC reads the midpoint voltage. In bright light the LDR resistance drops (e.g. 1 kΩ), so the voltage rises. In darkness the LDR resistance rises (e.g. 1 MΩ), so the voltage drops toward 0 V. GPIO 34 is input-only — ideal for ADC with no risk of accidental output drive.

## Code
```cpp
// LDR Light Intensity Meter
const int LDR_PIN = 34;

// Classify the ADC reading into light level bands
String classifyLight(int raw) {
  if      (raw < 400)  return "Dark";
  else if (raw < 1200) return "Dim";
  else if (raw < 2400) return "Moderate";
  else if (raw < 3500) return "Bright";
  else                 return "Very Bright";
}

void setup() {
  Serial.begin(115200);
  Serial.println("LDR Light Intensity Meter ready.");
}

void loop() {
  int raw = analogRead(LDR_PIN);   // 0–4095
  float voltage = raw * (3.3f / 4095.0f);

  Serial.print("ADC: "); Serial.print(raw);
  Serial.print("  |  ");
  Serial.print(voltage, 2); Serial.print(" V");
  Serial.print("  |  Level: ");
  Serial.println(classifyLight(raw));

  delay(400);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LDR** onto the canvas.
2. Connect LDR **output** to **GPIO34**.
3. Paste the code and click **Run**.
4. Adjust the LDR slider widget in the canvas to simulate different light levels.

## Expected Output
Serial Monitor:
```
LDR Light Intensity Meter ready.
ADC: 120   |  0.10 V  |  Level: Dark
ADC: 800   |  0.64 V  |  Level: Dim
ADC: 2100  |  1.69 V  |  Level: Moderate
ADC: 3200  |  2.58 V  |  Level: Bright
ADC: 4010  |  3.23 V  |  Level: Very Bright
```

## Expected Canvas Behavior
* Moving the LDR widget slider from dark to bright produces ascending ADC values and progresses through light level labels.
* The Serial Monitor updates at ~2.5 Hz with all three data columns.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `int raw = analogRead(LDR_PIN)` | Reads the midpoint voltage of the LDR voltage divider as a 12-bit integer. |
| `float voltage = raw * (3.3f / 4095.0f)` | Converts the raw count to volts. |
| `classifyLight(raw)` | A helper function using if/else-if thresholds to return a descriptive string. |
| `Serial.println(classifyLight(raw))` | Prints the category label on the same line as voltage and raw value. |

## Hardware & Safety Concept: Voltage Dividers with Sensors
An LDR is a resistor whose resistance decreases as light intensity increases. By placing it in series with a fixed resistor between 3V3 and GND, you create a **voltage divider**. The output voltage at the midpoint is: V = 3.3 × R_fixed ÷ (R_LDR + R_fixed). As R_LDR decreases (brighter light), the denominator shrinks, the voltage rises. This resistive-to-voltage conversion is how almost all resistive sensors (thermistors, LDRs, flex sensors, force-sensitive resistors) are read by microcontrollers — the sensor replaces one leg of the divider and varies the output voltage. Choosing the fixed resistor equal to the sensor's mid-scale resistance maximises sensitivity and linearity.

## Try This! (Challenges)
1. **Calibration**: Find the ADC values for your specific LDR in a dark room and under a desk lamp, then adjust the threshold constants.
2. **LED indicator**: Add 5 LEDs on GPIOs 12–16 and light a different number based on the light level category.
3. **Average smoothing**: Average 16 consecutive ADC reads before classifying to eliminate flicker noise.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Always reads very high (near 4095) | Missing pull-down resistor | Ensure the 10 kΩ resistor is between GPIO 34 and GND |
| Reading does not change with light | LDR legs not in correct divider positions | Verify LDR is between 3V3 and GPIO 34, resistor between GPIO 34 and GND |
| Serial output overlaps into one line | Missing `Serial.println` at end of print chain | Confirm last `Serial.print` is changed to `Serial.println` |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [31 - ESP32 Potentiometer ADC Read](31-esp32-potentiometer-adc-read.md)
- [34 - ESP32 LDR Automatic Dark Detector](34-esp32-ldr-automatic-dark-detector.md)
- [49 - ESP32 LDR Ambient Light LED Dimmer](49-esp32-ldr-ambient-light-led-dimmer.md)
