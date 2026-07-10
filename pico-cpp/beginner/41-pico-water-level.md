# 41 - Pico Water Level

Measure liquid levels in a container using an analog water level sensor.

## Goal
Learn how to read analog values from parallel-trace liquid sensors and print water depth levels to the Serial Monitor.

## What You Will Build
A water depth monitor:
- **Water Level Sensor (GP26)**: Connected to ADC0. As water covers more sensor traces, the analog output increases.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Water Level Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes (or depth module) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Water Level Sensor | VCC | 3.3V | Power supply |
| Water Level Sensor | OUT (Analog) | GP26 | Analog depth output |
| Water Level Sensor | GND | GND | Ground return |

## Code
```cpp
const int WATER_PIN = 26; // GP26 (ADC0)

void setup() {
  Serial.begin(9600);
  pinMode(WATER_PIN, INPUT);
  Serial.println("Water Depth Monitor Active");
}

void loop() {
  int waterLevel = analogRead(WATER_PIN);

  Serial.print("Raw Depth Reading: ");
  Serial.println(waterLevel);

  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Water Level Sensor** (represented by potentiometer) onto the canvas.
2. Connect Sensor: **VCC** to **3V3**, **OUT** to **GP26**, **GND** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the potentiometer representing water depth levels and monitor the Serial console.

## Expected Output

Terminal:
```
Water Depth Monitor Active
Raw Depth Reading: 250
Raw Depth Reading: 1850
```

## Expected Canvas Behavior
| Liquid Level | GP26 Read Value | Depth Status |
| --- | --- | --- |
| Dry (Empty) | < 300 | Tank Empty |
| Half Full | 1500–2200 | Medium Level |
| Overflow Warning | > 3000 | **Tank High Alert** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `analogRead(WATER_PIN)` | Reads the analog output voltage representing the conduction level across the immersion traces. |

## Hardware & Safety Concept: Trace Conduction
Water level sensor modules use a series of exposed, parallel copper traces. When immersed, the water acts as a conductor between the power lines and the signal lines, creating a variable resistance. Because water conductivity varies with mineral content (tap water conducts better than distilled water), calibration is required to map raw ADC readings to actual liquid depth.

## Try This! (Challenges)
1. **High Level Buzzer**: Sound an alarm on GP14 if the water depth exceeds 2800 to warn of container overflow.
2. **Pump Control Indicator**: Turn ON a green status LED on GP15 when the water level is safe, and flash a red LED if it drops below 500.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Reading stays stuck at 0 | Corrosion on probe | Tap water leaves mineral deposits on exposed traces over time. Clean the copper traces with isopropyl alcohol. |

## Mode Notes
This basic analog project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [39 - Pico Rain Sensor](39-pico-rain-sensor.md)
- [40 - Pico Soil Moisture](40-pico-soil-moisture.md)
