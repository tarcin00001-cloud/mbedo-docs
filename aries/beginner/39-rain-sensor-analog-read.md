# 39 - Rain Sensor Analog Read (GPIO Analog pin)

Measure and monitor rainfall wetness levels using an analog rain sensor connected to the VEGA ARIES v3 board.

## Goal
Understand how variable resistance sensors detect liquids, build a circuit to interface a liquid sensor board, and write code to invert and map readings into a percentage of wetness.

## What You Will Build
A rain sensor module is connected to analog input pin `ADC1` (`GP27`). The sensor plate's resistance changes when water droplets land on it. The board reads the resulting voltage changes, maps them so that dry is 0% and fully wet is 100%, and prints the wetness level to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Rain Sensor & Driver Module | `rain_sensor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Rain Sensor Board | VCC | 3.3V | Red | Power connection |
| Rain Sensor Board | AO (Analog Out) | ADC1 (GP27) | Yellow | Analog wetness level signal |
| Rain Sensor Board | GND | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The rain sensor board uses a comparative driver module. Hook up the Analog Output (AO) pin to GP27 (ADC1) to read the gradual wetness curve. The Digital Output (DO) pin can be left disconnected.

## Code
```cpp
// Rain Sensor Analog Read - VEGA ARIES v3
const int RAIN_PIN = GP27; // ADC1

void setup() {
  // Initialize serial communication at 115200 bps
  Serial.begin(115200);
}

void loop() {
  // Read raw analog value from the sensor
  int rawValue = analogRead(RAIN_PIN);
  
  // Map raw value: Dry (4095) is 0% wetness, fully wet (0) is 100% wetness
  int wetness = map(rawValue, 4095, 0, 0, 100);
  
  // Constrain the result to guarantee it stays in the 0-100% range
  if (wetness < 0) {
    wetness = 0;
  }
  if (wetness > 100) {
    wetness = 100;
  }

  // Print results
  Serial.print("Raw Sensor Value: ");
  Serial.print(rawValue);
  Serial.print(" | Wetness Level: ");
  Serial.print(wetness);
  Serial.println("%");

  // Read wetness level every 1 second
  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board and a **Rain Sensor** onto the canvas.
2. Wire the Rain Sensor's VCC to **3.3V**, AO to **GP27 (ADC1)**, and GND to **GND**.
3. Paste the code into the editor.
4. Click **Run**.
5. Adjust the water level slider on the rain sensor widget and observe the Serial Monitor.

## Expected Output
Serial Monitor:
```
System Initialized.
Raw Sensor Value: 4095 | Wetness Level: 0%
Raw Sensor Value: 2048 | Wetness Level: 50%
Raw Sensor Value: 0 | Wetness Level: 100%
```

## Expected Canvas Behavior
* The Serial Monitor prints the mapped wetness level from 0% (completely dry) to 100% (soaked) as the rain sensor slider is modified.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `const int RAIN_PIN = GP27;` | Configures GP27 (ADC1) as the input. |
| `map(rawValue, 4095, 0, 0, 100)` | Inverts the scale: a high value (low conductivity, dry) maps to 0%, while a low value (high conductivity, wet) maps to 100%. |
| `if (wetness < 0) ...` | Checks boundaries to prevent noise from showing negative or over-100 values. |

## Hardware & Safety Concept: Resistance Variation and Electrolytic Corrosion
* **Resistance Variation**: Rain sensors have interleaved copper traces that behave as a variable resistor. When dry, resistance is extremely high (megohms). Water droplets bridge the traces, establishing conductive paths that lower the resistance.
* **Electrolytic Corrosion**: Passing DC current through wet sensor traces causes electrolysis, causing rapid oxidation and degradation of the copper traces. To extend sensor life, only power the sensor when reading (e.g. using a digital pin as a power source) and keep current minimal.

## Try This! (Challenges)
1. **Rain Alarm**: Turn ON the onboard buzzer (`GPIO 14`) if the wetness level exceeds 30%.
2. **Sensor Power Gate**: Wire the rain sensor's VCC to a digital GPIO pin (e.g., `GPIO 15`) and turn it HIGH only right before reading, then LOW immediately after, to prevent trace corrosion.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Wetness level is stuck at 0% or 100% | Loose connections or wrong output pin | Verify the wire is connected to the AO (Analog Output) pin on the module, not the DO (Digital Output) pin |
| Sensor behaves in reverse | Incorrect mapping formula | Ensure your map parameters are set as `map(rawValue, 4095, 0, 0, 100)` so that lower values produce higher percentages |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [38 - Piezo vibration intensity level print](38-piezo-vibration-intensity-level-print.md) (Previous project)
- [40 - Soil moisture sensor level print](40-soil-moisture-sensor-level-print.md) (Next project)
