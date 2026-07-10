# 43 - Pico MQ-2 Gas Sensor

Measure ambient smoke and gas concentrations using an MQ-2 sensor.

## Goal
Learn how to interface analog gas sensors with the Pico's ADC and log air quality metrics to the Serial Monitor.

## What You Will Build
A gas safety monitor:
- **MQ-2 Gas Sensor (GP26)**: Connected to ADC0. Exposing the sensor to gas increases the analog reading printed to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MQ-2 Gas Sensor | `gas_sensor` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MQ-2 Sensor | VCC | 5V (or VBUS) | High power heating supply |
| MQ-2 Sensor | AO (Analog Out) | GP26 (ADC0) | Analog gas signal |
| MQ-2 Sensor | GND | GND | Ground return |

## Code
```cpp
const int GAS_PIN = 26; // GP26 (ADC0)

void setup() {
  Serial.begin(9600);
  pinMode(GAS_PIN, INPUT);
  Serial.println("MQ-2 Gas Monitor Online");
}

void loop() {
  int gasLevel = analogRead(GAS_PIN);

  Serial.print("Raw Gas Value: ");
  Serial.println(gasLevel);

  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **MQ-2 Gas Sensor** onto the canvas.
2. Connect Gas Sensor: **VCC** to **5V** (or **VBUS**), **AO** to **GP26**, **GND** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the gas PPM level slider on canvas and observe the changing Serial logs.

## Expected Output

Terminal:
```
MQ-2 Gas Monitor Online
Raw Gas Value: 450
Raw Gas Value: 1850
```

## Expected Canvas Behavior
| Gas PPM level slider | GP26 Read Value | Air Quality Status |
| --- | --- | --- |
| Minimum (Clean Air) | 300–600 | Normal / Clean |
| Moderate Smoke | 1000–2200 | Warning / Ventilate |
| High Gas (Leak) | > 2800 | **CRITICAL ALERT** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `analogRead(GAS_PIN)` | Returns the analog voltage output from the gas sensor heater-bridge circuit. |

## Hardware & Safety Concept: Sensor Warm-up Cycle
MQ-series gas sensors contain an internal heating element wrapped around a tin dioxide ($SnO_2$) semiconductor layer. When hot, oxygen is adsorbed on the semiconductor surface, restricting current flow. In the presence of combustible gases (like LPG, propane, or hydrogen), the oxygen concentration drops, increasing conduction. Because the heater requires a stable temperature, the sensor must **warm up** for at least 24 hours on first use, and 1-2 minutes on boot, before readings stabilize.

## Try This! (Challenges)
1. **Gas Alarm**: Sound a warning buzzer on GP14 if the gas level exceeds 1500.
2. **Status Light**: Turn ON a green LED on GP15 when gas levels are below 1000, and a red LED when above.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sensor gets very warm to the touch | Normal operation | The internal heating coil runs at high temperatures. This is expected behavior and necessary for gas detection. |

## Mode Notes
This basic analog project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [31 - Pico Potentiometer ADC](31-pico-potentiometer-adc.md)
- [50 - Pico Sound Alarm](50-pico-sound-alarm.md)
