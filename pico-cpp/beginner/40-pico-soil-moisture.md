# 40 - Pico Soil Moisture

Measure soil moisture levels using an analog soil moisture probe.

## Goal
Learn how to interface resistive soil moisture probes and read varying analog moisture indices using the Pico's ADC.

## What You Will Build
A soil humidity checker:
- **Soil Moisture Sensor (GP26)**: Connected to ADC0. Toggling the moisture level updates the raw ADC value printed to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Soil Moisture Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes (or soil probe module) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Moisture Sensor | VCC | 3.3V | Power supply |
| Moisture Sensor | AO (Analog Out) | GP26 | Analog moisture level |
| Moisture Sensor | GND | GND | Ground return |

## Code
```cpp
const int SOIL_PIN = 26; // GP26 (ADC0)

void setup() {
  Serial.begin(9600);
  pinMode(SOIL_PIN, INPUT);
  Serial.println("Soil Moisture Sensor Online");
}

void loop() {
  int rawMoisture = analogRead(SOIL_PIN);

  Serial.print("Raw Moisture Level: ");
  Serial.println(rawMoisture);

  delay(1000); // Sample once per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Soil Moisture Sensor** (represented by potentiometer) onto the canvas.
2. Connect Moisture Sensor: **VCC** to **3V3**, **AO** to **GP26**, **GND** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the potentiometer representing moisture levels and observe the Serial logs.

## Expected Output

Terminal:
```
Soil Moisture Sensor Online
Raw Moisture Level: 3500
Raw Moisture Level: 1200
```

## Expected Canvas Behavior
| Moisture Slider Position | GP26 Read Value | Soil Condition |
| --- | --- | --- |
| Far Left (Dry) | > 3500 | Dry Soil (Needs water) |
| Center | 1800–2800 | Moist Soil |
| Far Right (Wet) | < 1000 | Wet Soil / Water |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `analogRead(SOIL_PIN)` | Reads the analog voltage output from the resistive soil probe voltage divider circuit. |

## Hardware & Safety Concept: Sensor Corrosion
Resistive soil moisture probes pass current between two exposed metal leads stuck in the soil. Because water conducts electricity, wetter soil decreases the resistance between the leads. However, running current continuously through the probe in wet soil causes **electrochemical corrosion** (electrolysis), which eats away the metal traces. On real hardware, power the sensor from a GPIO pin and turn it ON only during measurement cycles.

## Try This! (Challenges)
1. **Irrigation Indicator**: Turn ON an external Red LED on GP15 if the moisture level drops below 1500 (indicating dry soil).
2. **Moisture Alarm**: Sound a warning buzzer on GP10 if the soil is critically dry.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Readings jump randomly | Soil contact issue | Real probes must be pressed firmly into the dirt to ensure a solid electrical interface. |

## Mode Notes
This basic analog project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [31 - Pico Potentiometer ADC](31-pico-potentiometer-adc.md)
- [41 - Pico Water Level](41-pico-water-level.md)
