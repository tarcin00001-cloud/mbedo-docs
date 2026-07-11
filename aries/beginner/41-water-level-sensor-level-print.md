# 41 - Water Level Sensor Level Print (GPIO Analog pin)

Measure and print liquid depth levels using an analog water level sensor connected to the VEGA ARIES v3 board.

## Goal
Understand the electrical principles of liquid conductivity sensors, wire a water level sensor to the ARIES board's ADC, and implement calibration mapping in code to print water level percentages.

## What You Will Build
An analog water level sensor is connected to analog input pin `ADC2` (`GP28`). As water rises along the exposed metal traces of the sensor, it changes the output voltage. The board reads this voltage, applies a calibrated mapping to convert it to a level (0% to 100%), and prints it to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Water Level Sensor | `water_level` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Water Level Sensor | + (VCC) | 3.3V | Red | Power connection |
| Water Level Sensor | S (Signal) | ADC2 (GP28) | Yellow | Analog signal output |
| Water Level Sensor | - (GND) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Keep the water level sensor's top electronic circuit dry. Only immerse the lower parallel metallic traces into the liquid. Wiring is direct with no external pull-down resistor required.

## Code
```cpp
// Water Level Sensor Print - VEGA ARIES v3
const int WATER_PIN = GP28; // ADC2

void setup() {
  // Initialize serial communication at 115200 bps
  Serial.begin(115200);
}

void loop() {
  // Read raw analog value from the sensor
  int rawValue = analogRead(WATER_PIN);
  
  // Map raw value: Dry (0) is 0%, fully submerged (3000) is 100%
  // 3000 is used instead of 4095 as the sensor typically saturates early
  int waterLevel = map(rawValue, 0, 3000, 0, 100);
  
  // Clamp results to keep values strictly within 0% to 100%
  if (waterLevel < 0) {
    waterLevel = 0;
  }
  if (waterLevel > 100) {
    waterLevel = 100;
  }
  
  // Print values to the Serial Monitor
  Serial.print("Water ADC Raw: ");
  Serial.print(rawValue);
  Serial.print(" | Water Level: ");
  Serial.print(waterLevel);
  Serial.println("%");
  
  // Sample the water level every 1 second
  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board and a **Water Level Sensor** onto the canvas.
2. Wire the Water Level Sensor's VCC to **3.3V**, Signal (S) to **GP28 (ADC2)**, and GND (-) to **GND**.
3. Paste the code into the editor.
4. Click **Run**.
5. Adjust the water depth slider on the sensor widget and view the Serial Monitor.

## Expected Output
Serial Monitor:
```
System Initialized.
Water ADC Raw: 0 | Water Level: 0%
Water ADC Raw: 1500 | Water Level: 50%
Water ADC Raw: 3000 | Water Level: 100%
```

## Expected Canvas Behavior
* The Serial Monitor output dynamically prints the mapped water depth from 0% (dry) to 100% (fully submerged) as you adjust the slider on the sensor component.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `const int WATER_PIN = GP28;` | Defines the signal input pin on the ARIES board. |
| `map(rawValue, 0, 3000, 0, 100)` | Maps the analog value range (calibrated between 0 and 3000 for standard submersion limits) to 0–100%. |
| `if (waterLevel < 0) ...` | Clamps the output value to stay within logical bounds. |

## Hardware & Safety Concept: Trace Resistance and Liquid Conductivity
* **Trace Resistance**: The water level sensor operates by using a series of parallel exposed traces connected to ground and signal lines. When water bridges these traces, it acts as a variable resistor whose value decreases as more sensor area is submerged.
* **Liquid Conductivity**: The sensor relies on the electrical conductivity of water. Pure distilled water is non-conductive and will not register a reading. Tap water, rainwater, or sewage water containing ions are highly conductive, providing clear analog feedback.

## Try This! (Challenges)
1. **Overflow Alarm**: Turn ON the onboard buzzer (`GPIO 14`) if the water level exceeds 85% to signal a high-water/overflow warning.
2. **LED Bar Graph Indicator**: Wire three external LEDs to indicate low, medium, and high levels (e.g. green LED for <50%, yellow LED for 50-80%, red LED for >80%).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Reads 100% when dry | Short circuit between traces | Check if there is residual water or conductive debris bridging the sensor's metal traces. Dry the sensor with a paper towel |
| Reads 0% even when submerged | Distilled water used or VCC disconnected | Verify the + (VCC) pin is wired to 3.3V and the water contains natural trace minerals or a tiny pinch of salt to enable conductivity |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [40 - Soil moisture sensor level print](40-soil-moisture-sensor-level-print.md) (Previous project)
- [42 - MQ-2 Gas sensor level print](42-mq-2-gas-sensor-level-print.md) (Next project)
