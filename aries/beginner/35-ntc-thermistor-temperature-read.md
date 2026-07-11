# 35 - NTC Thermistor Temperature Read (GPIO Analog pin)

Measure and display ambient temperature using an NTC Thermistor connected to the VEGA ARIES v3 board.

## Goal
Understand how a Negative Temperature Coefficient (NTC) thermistor works, build a voltage divider circuit, and apply the B-coefficient Steinhart-Hart formula in code to output temperature in Celsius.

## What You Will Build
An NTC thermistor (10 kΩ nominal) is connected in series with a 10 kΩ resistor on analog input pin `ADC0` (`GP26`). The board measures the voltage, calculates the thermistor's resistance, computes temperature using a logarithmic equation, and prints the Celsius temperature to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| NTC Thermistor (10 kΩ) | `thermistor` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Thermistor | Pin 1 | 3.3V | Red | Power connection |
| Thermistor | Pin 2 | ADC0 (GP26) | Yellow | Analog input signal (at voltage divider midpoint) |
| 10 kΩ Resistor | Pin 1 | ADC0 (GP26) | Yellow | Connected to Thermistor Pin 2 |
| 10 kΩ Resistor | Pin 2 | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Ensure the 10 kΩ series resistor is connected between GP26 (ADC0) and GND. This forms the lower leg of the voltage divider, while the NTC Thermistor is connected between 3.3V and GP26.

## Code
```cpp
// NTC Thermistor Temperature Read - VEGA ARIES v3
#include <math.h>

const int NTC_PIN = GP26; // ADC0

void setup() {
  // Initialize serial communication
  Serial.begin(115200);
}

void loop() {
  // Read raw analog value
  int rawValue = analogRead(NTC_PIN);
  
  // Guard to prevent division by zero or negative log inputs
  if (rawValue > 0 && rawValue < 4095) {
    // Calculate thermistor resistance: R_NTC = R_series * (ADC_max / ADC_val - 1)
    float resistance = 10000.0 * (4095.0 / (float)rawValue - 1.0);
    
    // B-parameter equation: 1/T = 1/T0 + (1/B) * ln(R / R0)
    // T0 = 25C (298.15K), R0 = 10k Ohm, B = 3950 K
    float tempK = 1.0 / (1.0 / 298.15 + (1.0 / 3950.0) * log(resistance / 10000.0));
    
    // Convert Kelvin to Celsius
    float tempC = tempK - 273.15;
    
    // Print the output
    Serial.print("Raw ADC: ");
    Serial.print(rawValue);
    Serial.print(" | Resistance: ");
    Serial.print(resistance);
    Serial.print(" Ohm | Temp: ");
    Serial.print(tempC);
    Serial.println(" C");
  } else {
    Serial.println("Error: Sensor input out of valid range.");
  }
  
  // Sample temperature every 1 second
  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, an **NTC Thermistor**, a **10 kΩ Resistor**, and a **Breadboard** onto the canvas.
2. Wire the NTC Thermistor voltage divider to **GP26 (ADC0)**.
3. Paste the code into the editor.
4. Click **Run**.
5. Adjust the temperature slider on the thermistor widget and view the values in the Serial Monitor.

## Expected Output
Serial Monitor:
```
System Initialized.
Raw ADC: 2048 | Resistance: 10000.00 Ohm | Temp: 25.00 C
Raw ADC: 2890 | Resistance: 4169.55 Ohm | Temp: 45.10 C
```

## Expected Canvas Behavior
* The Serial Monitor prints the updated temperature as you adjust the slider on the thermistor component.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `#include <math.h>` | Includes standard math library for the log function. |
| `4095.0 / (float)rawValue - 1.0` | Formula to calculate the ratio of thermistor resistance to the series resistor. |
| `1.0 / (1.0 / 298.15 + (1.0 / ...` | B-coefficient formula to compute absolute temperature in Kelvin. |

## Hardware & Safety Concept: Negative Temperature Coefficient and Linearization
* **Negative Temperature Coefficient (NTC)**: The resistance of an NTC thermistor decreases as temperature increases. This property is due to semi-conductive materials where heat frees charge carriers.
* **Linearization**: Because the resistance-to-temperature relationship of a thermistor is highly non-linear (logarithmic), software linearization (like the Steinhart-Hart equation) is required to convert resistance readings into standard temperature units.

## Try This! (Challenges)
1. **High Temp Alarm**: Turn ON the warning LED (`GPIO 15`) if the temperature exceeds 30°C.
2. **Fahrenheit Conversion**: Modify the code to calculate and display the temperature in Fahrenheit ($F = C \times 1.8 + 32$).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Temperature is extremely high or negative | Wrong resistor size or inverted voltage divider | Make sure you are using a 10 kΩ resistor and that the thermistor is connected between 3.3V and the ADC pin |
| Serial Monitor outputs NaN | Division by zero or log of a negative number | Check that the raw ADC reading is strictly greater than 0 and less than 4095 to prevent domain errors in math calculations |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [34 - LDR Automatic Dark Detector](34-ldr-automatic-dark-detector.md) (Previous project)
- [36 - PIR motion sensor trigger](36-pir-motion-sensor-trigger.md) (Next project)
