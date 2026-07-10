# 40 - ESP32 Soil Moisture Analog Level Print

Read the analog output of a capacitive (or resistive) soil moisture sensor and print the moisture percentage and a descriptive label to the Serial Monitor.

## Goal
Learn how a soil moisture sensor converts soil water content into an analog voltage and how to map the ADC reading into a 0–100% scale with descriptive soil condition labels.

## What You Will Build
A soil moisture sensor module with its analog output connected to GPIO 34. The code reads the ADC, maps it to a moisture percentage, classifies the soil as Dry, Low, Moderate, Moist, or Wet, and logs the result every second.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Soil Moisture Sensor Module | `soil_sensor` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Soil Sensor Module | VCC | 3V3 | Red | Module supply voltage |
| Soil Sensor Module | GND | GND | Black | Module ground |
| Soil Sensor Module | AO (analog out) | GPIO34 | Yellow | Analog moisture signal |

> **Wiring tip:** Capacitive soil sensors (preferred) output HIGH voltage (~3.0 V) when dry and lower voltage (~1.2 V) when fully wet — the inverse of resistive sensors. Resistive sensors degrade over time in wet soil; capacitive sensors are more durable. For resistive modules: HIGH voltage = dry, LOW voltage = wet. For capacitive modules: HIGH voltage = dry (no dielectric), LOW voltage = wet (water changes dielectric constant). Calibrate your specific sensor by measuring the ADC value in completely dry soil and saturated soil, then set those as your mapping bounds.

## Code
```cpp
// Soil Moisture Analog Level Print
const int SOIL_PIN = 34;

// Calibration — adjust these values for your specific sensor in your soil
const int DRY_VAL  = 3500;   // ADC reading in completely dry soil
const int WET_VAL  = 1200;   // ADC reading in saturated soil

String classifySoil(int pct) {
  if      (pct < 15) return "DRY   — needs water urgently";
  else if (pct < 35) return "LOW   — water soon";
  else if (pct < 55) return "MODERATE — adequate";
  else if (pct < 75) return "MOIST — good moisture";
  else               return "WET   — well saturated";
}

void setup() {
  Serial.begin(115200);
  Serial.println("Soil Moisture Sensor ready.");
}

void loop() {
  int raw = analogRead(SOIL_PIN);

  // Map ADC to percentage: DRY_VAL = 0%, WET_VAL = 100%
  int moisture = map(raw, DRY_VAL, WET_VAL, 0, 100);
  moisture = constrain(moisture, 0, 100);   // Clamp to valid range

  Serial.print("ADC: "); Serial.print(raw);
  Serial.print("  |  Moisture: "); Serial.print(moisture);
  Serial.print("%  |  Soil: ");
  Serial.println(classifySoil(moisture));

  delay(1000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Soil Moisture Sensor** onto the canvas.
2. Connect Sensor **AO** to **GPIO34**.
3. Paste the code and click **Run**.
4. Drag the soil sensor slider widget — observe moisture percentage and label changing.

## Expected Output
Serial Monitor:
```
Soil Moisture Sensor ready.
ADC: 3480  |  Moisture: 1%    |  Soil: DRY   — needs water urgently
ADC: 2600  |  Moisture: 39%   |  Soil: MODERATE — adequate
ADC: 1800  |  Moisture: 74%   |  Soil: MOIST — good moisture
ADC: 1250  |  Moisture: 99%   |  Soil: WET   — well saturated
```

## Expected Canvas Behavior
* Moving the soil sensor slider from dry to wet decreases the ADC value and increases the moisture percentage.
* The soil condition label updates at each slider position.
* Serial Monitor logs every 1 second.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `const int DRY_VAL = 3500` | The ADC value when the sensor is in completely dry soil — your baseline for 0%. |
| `const int WET_VAL = 1200` | The ADC value when the sensor is in saturated soil — your baseline for 100%. |
| `map(raw, DRY_VAL, WET_VAL, 0, 100)` | Linearly interpolates the ADC reading into a 0–100% moisture scale. |
| `constrain(moisture, 0, 100)` | Clamps the result to 0–100 in case ADC readings exceed calibration bounds. |
| `classifySoil(moisture)` | Returns a human-readable label for the moisture band. |

## Hardware & Safety Concept: Capacitive Soil Moisture Sensing
Soil moisture sensors work by measuring the electrical property of soil that changes with water content. Resistive sensors measure **conductivity** (water conducts electricity). Capacitive sensors measure **dielectric permittivity** (water has a dielectric constant of ~80, much higher than dry soil at ~4). As soil absorbs water, the dielectric constant increases, which changes the capacitance of the sensor electrodes, which shifts the oscillation frequency of the sensor's internal RC circuit. This frequency shift is converted to an output voltage. Capacitive sensors are preferred because they: do not corrode (no current flows through the soil), do not polarise the soil, and measure volumetric water content rather than salt concentration.

## Try This! (Challenges)
1. **Irrigation relay**: Add a relay on GPIO 5. Automatically turn it on (to run a water pump) when moisture drops below 20%, and off above 50%.
2. **Calibration mode**: Print raw ADC values for 30 seconds in dry soil and 30 seconds in wet soil to determine your `DRY_VAL` and `WET_VAL`.
3. **Trend detection**: Track the last three readings; if moisture is decreasing across all three, print "Soil drying — water soon."

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Moisture always shows 0% or 100% | Calibration values wrong for your sensor | Measure actual dry and wet ADC values and update `DRY_VAL` and `WET_VAL` |
| ADC reading is stable despite soil changes | AO pin wired to DO (digital out) instead of AO | Use the module's AO pin, not DO, for analog output |
| Readings fluctuate rapidly | Loose sensor-to-GPIO wire | Re-seat the AO wire; add a 100 nF capacitor from GPIO 34 to GND |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [39 - ESP32 Rain Sensor Digital Alert](39-esp32-rain-sensor-digital-alert.md)
- [41 - ESP32 Water Level Analog Level Print](41-esp32-water-level-analog-level-print.md)
- [34 - ESP32 LDR Automatic Dark Detector](34-esp32-ldr-automatic-dark-detector.md)
