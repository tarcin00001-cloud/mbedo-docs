# 35 - Pico NTC Thermistor

Measure ambient temperature levels using an NTC Thermistor analog sensor.

## Goal
Learn how to read analog voltages from temperature-sensitive resistors and calculate log read-outs to the Serial Monitor.

## What You Will Build
An analog thermometer:
- **NTC Thermistor (GP26)**: Connected to ADC0. Its raw analog voltage output represents ambient thermal changes.
- **Serial Monitor**: Displays the raw ADC reading and calculated temperature trends.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| NTC Thermistor | `thermistor` | Yes | Yes |
| 10k-ohm Resistor | `resistor` | Optional | Yes (voltage divider) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Thermistor | Pin 1 | 3.3V | Power supply |
| Thermistor | Pin 2 | GP26 | Analog input (wire 10k resistor from GP26 to GND) |
| Thermistor | Pin 2 | GND | Ground return via resistor |

## Code
```cpp
const int NTC_PIN = 26; // GP26 (ADC0)

void setup() {
  Serial.begin(9600);
  pinMode(NTC_PIN, INPUT);
  Serial.println("Thermistor Temperature Monitor Active");
}

void loop() {
  int rawValue = analogRead(NTC_PIN);

  // Convert raw value to a simplified temperature metric (0-100 scale)
  int tempIndex = rawValue / 40; 

  Serial.print("Raw ADC: ");
  Serial.print(rawValue);
  Serial.print(" | Relative Temp Index: ");
  Serial.println(tempIndex);

  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **NTC Thermistor** onto the canvas.
2. Connect Thermistor: **Pin 1** to **3V3**, **Pin 2** to **GP26** (with 10k resistor to GND).
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the Thermistor slider on canvas and observe the printed temperature indices.

## Expected Output

Terminal:
```
Thermistor Temperature Monitor Active
Raw ADC: 1600 | Relative Temp Index: 40
Raw ADC: 2400 | Relative Temp Index: 60
```

## Expected Canvas Behavior
| Temp Slider Position | GP26 Read Value | Temp Index |
| --- | --- | --- |
| Cold | < 1200 | < 30 |
| Normal | 1600–2200 | 40–55 |
| Hot | > 2800 | > 70 |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `rawValue / 40` | Computes a simplified linear estimation scale for raw ADC levels to show trend movements. |

## Hardware & Safety Concept: Negative Temperature Coefficient (NTC)
NTC stands for **Negative Temperature Coefficient**. As the temperature of an NTC thermistor increases, its internal resistance decreases. Consequently, the output voltage from the voltage divider increases. Conversely, **PTC** (Positive Temperature Coefficient) thermistors increase in resistance as temperature goes up.

## Try This! (Challenges)
1. **Overheat Alert**: Turn ON an external Red LED on GP15 if the `tempIndex` exceeds 65.
2. **Ice Alert**: Flash a Blue LED on GP14 if the `tempIndex` drops below 25.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Temperature reading stays constant | Sensor disconnected | Check wiring connections; ensure GP26 is wired to the junction of the thermistor and the 10k resistor. |

## Mode Notes
This basic analog monitoring project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [31 - Pico Potentiometer ADC](31-pico-potentiometer-adc.md)
- [34 - Pico LDR Dark Detector](34-pico-ldr-dark-detector.md)
