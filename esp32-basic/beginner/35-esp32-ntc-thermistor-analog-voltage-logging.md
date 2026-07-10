# 35 - ESP32 NTC Thermistor Analog Voltage Logging

Read the analog voltage from an NTC thermistor voltage divider and convert it to a temperature in degrees Celsius using the Steinhart-Hart equation.

## Goal
Learn how a Negative Temperature Coefficient (NTC) thermistor changes resistance with temperature, how to read it with a voltage divider, and how to apply the Steinhart-Hart equation to convert ADC readings into calibrated temperature values.

## What You Will Build
An NTC thermistor and a 10 kΩ fixed resistor forming a voltage divider on GPIO 34. The code reads the ADC, computes resistance, and uses the Steinhart-Hart equation to print temperature in °C every 500 ms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| NTC Thermistor (10 kΩ at 25 °C) | `thermistor` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NTC Thermistor | Leg 1 | 3V3 | Red | Supply voltage (top of divider) |
| NTC Thermistor | Leg 2 | GPIO34 | Yellow | Divider midpoint — ADC input |
| 10 kΩ Resistor | Leg 1 | GPIO34 | White | Pull-down leg of divider |
| 10 kΩ Resistor | Leg 2 | GND | Black | Ground reference |

> **Wiring tip:** The thermistor is not polarised — either leg can be the top or bottom of the divider. Keep the wires short to minimise capacitive noise. Add a 100 nF ceramic capacitor between GPIO 34 and GND if ADC readings are noisy. The 10 kΩ resistor must match the thermistor's nominal resistance at 25 °C for best sensitivity across the temperature range.

## Code
```cpp
// NTC Thermistor Voltage Logger — Steinhart-Hart temperature calculation
#include <math.h>

const int   ADC_PIN    = 34;
const float R_FIXED    = 10000.0;  // Fixed resistor in the divider (10 kΩ)
const float R_NOMINAL  = 10000.0;  // Thermistor resistance at 25 °C (10 kΩ)
const float T_NOMINAL  = 25.0;     // Reference temperature in °C
const float B_COEFF    = 3950.0;   // Beta coefficient (check thermistor datasheet)
const int   ADC_MAX    = 4095;

float readTemperature() {
  int raw = analogRead(ADC_PIN);
  float voltage = raw * (3.3f / (float)ADC_MAX);

  // Compute thermistor resistance from voltage divider equation
  // V_out = 3.3 * R_FIXED / (R_NTC + R_FIXED)  →  R_NTC = R_FIXED * (3.3/V_out - 1)
  float rThermistor = R_FIXED * (3.3f / voltage - 1.0f);

  // Steinhart-Hart simplified B-parameter equation
  // 1/T = 1/T0 + (1/B) * ln(R/R0)    (temperatures in Kelvin)
  float steinhart = rThermistor / R_NOMINAL;
  steinhart = log(steinhart);                         // ln(R/R_NOMINAL)
  steinhart /= B_COEFF;                               // ÷ B coefficient
  steinhart += 1.0f / (T_NOMINAL + 273.15f);          // + 1/T_NOMINAL (K)
  float tempK = 1.0f / steinhart;
  return tempK - 273.15f;   // Convert Kelvin to Celsius
}

void setup() {
  Serial.begin(115200);
  Serial.println("NTC Thermistor Logger ready.");
}

void loop() {
  int raw    = analogRead(ADC_PIN);
  float temp = readTemperature();

  Serial.print("ADC Raw: "); Serial.print(raw);
  Serial.print("  |  Temperature: "); Serial.print(temp, 1);
  Serial.println(" °C");

  delay(500);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Thermistor** onto the canvas.
2. Connect Thermistor **signal output** to **GPIO34**.
3. Paste the code and click **Run**.
4. Adjust the thermistor slider widget — observe temperature changing in the Serial Monitor.

## Expected Output
Serial Monitor:
```
NTC Thermistor Logger ready.
ADC Raw: 2048  |  Temperature: 25.0 °C
ADC Raw: 2580  |  Temperature: 18.2 °C
ADC Raw: 1560  |  Temperature: 34.7 °C
ADC Raw: 980   |  Temperature: 46.5 °C
```

## Expected Canvas Behavior
* Moving the thermistor slider upward (hotter) decreases resistance, raises voltage, increases ADC reading, and increases temperature readout.
* Moving the slider downward (cooler) increases resistance, lowers voltage, decreases ADC reading, and decreases temperature readout.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `float rThermistor = R_FIXED * (3.3f / voltage - 1.0f)` | Rearranges the voltage divider equation to compute thermistor resistance from the measured voltage. |
| `steinhart = log(steinhart)` | Computes the natural logarithm of the resistance ratio — part of the Steinhart-Hart formula. |
| `steinhart += 1.0f / (T_NOMINAL + 273.15f)` | Adds 1/T₀ in Kelvin — the reference temperature term. |
| `float tempK = 1.0f / steinhart` | Inverts the result to get temperature in Kelvin. |
| `return tempK - 273.15f` | Converts Kelvin to Celsius by subtracting 273.15. |

## Hardware & Safety Concept: NTC Thermistors and the Steinhart-Hart Equation
An NTC (Negative Temperature Coefficient) thermistor is a semiconductor resistor whose resistance decreases exponentially as temperature rises. Unlike a linear resistor, its resistance-temperature curve follows the relationship: R = R₀ × e^(B × (1/T − 1/T₀)), where B is the **beta coefficient** (a material constant typically 3000–5000 K for common thermistors). The Steinhart-Hart equation is the inverse form: it takes resistance and outputs temperature. NTC thermistors are inexpensive, sensitive, and fast-responding — widely used in battery chargers, refrigerators, HVAC systems, and medical devices. Their limitation is a non-linear response that requires the B-coefficient calculation (or a full three-term Steinhart-Hart polynomial) for accuracy across a wide range.

## Try This! (Challenges)
1. **Fahrenheit output**: Add `float tempF = temp * 9.0f/5.0f + 32.0f` and print both scales.
2. **Temperature alarm**: Print "OVER TEMPERATURE" if the reading exceeds 40 °C.
3. **Moving average**: Average 10 ADC readings before computing temperature to reduce noise.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Temperature reads very high (90+ °C) | Thermistor and fixed resistor swapped | Move thermistor to the top (3V3 side) and resistor to the bottom (GND side) |
| Temperature reads −50 °C or extreme values | ADC reads 0 or 4095 | Verify both divider components are correctly connected to GPIO 34 |
| Temperature drifts even in stable conditions | ADC noise | Average 8+ readings before computing; add a 100 nF decoupling capacitor |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [31 - ESP32 Potentiometer ADC Read](31-esp32-potentiometer-adc-read.md)
- [33 - ESP32 LDR Light Intensity Meter](33-esp32-ldr-light-intensity-meter.md)
- [34 - ESP32 LDR Automatic Dark Detector](34-esp32-ldr-automatic-dark-detector.md)
