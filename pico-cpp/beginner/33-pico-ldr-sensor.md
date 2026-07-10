# 33 - Pico LDR Sensor

Measure ambient light intensity using a Light Dependent Resistor (LDR) analog sensor.

## Goal
Learn how to read analog inputs from variable-resistance sensor circuits and print raw light levels to the Serial Monitor.

## What You Will Build
An ambient light meter:
- **LDR Sensor (GP26)**: Connected to ADC0. It outputs higher raw values under bright light and lower values in darkness.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LDR (Photoresistor) | `photoresistor` | Yes | Yes |
| 10k-ohm Resistor | `resistor` | Optional | Yes (voltage divider) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| LDR | Pin 1 | 3.3V | Power supply |
| LDR | Pin 2 (Signal) | GP26 | Connects to GP26 (wire 10k resistor from GP26 to GND) |
| LDR | Pin 2 | GND | Ground return via resistor |

## Code
```cpp
const int LDR_PIN = 26; // GP26 (ADC0)

void setup() {
  Serial.begin(9600);
  pinMode(LDR_PIN, INPUT);
  Serial.println("LDR Ambient Light Monitor Active");
}

void loop() {
  int lightLevel = analogRead(LDR_PIN); // Reads 0 to 4095

  Serial.print("Raw Light Level: ");
  Serial.println(lightLevel);

  delay(500);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **LDR Sensor** onto the canvas.
2. Connect LDR: **Pin 1** to **3V3**, **Pin 2** to **GP26** (wire a 10k resistor from GP26 to GND).
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the LDR light slider on canvas and observe the changing Serial logs.

## Expected Output

Terminal:
```
LDR Ambient Light Monitor Active
Raw Light Level: 500
Raw Light Level: 3500
```

## Expected Canvas Behavior
| Light Slider Position | GP26 Read Value | Light Condition |
| --- | --- | --- |
| Darkest (Min) | < 800 | Dark / Night |
| Room Light | 1500–2500 | Normal |
| Brightest (Max) | > 3500 | Bright Sunlight |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `analogRead(LDR_PIN)` | Reads the analog voltage output by the LDR voltage divider circuit. |

## Hardware & Safety Concept: Voltage Divider Circuits
LDRs are variable resistors whose resistance decreases as light levels increase. Microcontrollers cannot measure resistance directly; they only measure voltage. By placing a fixed resistor (10k ohms) in series with the LDR to form a **voltage divider**, the changing resistance is converted into a proportional voltage signal (0V to 3.3V) that the ADC can read.

## Try This! (Challenges)
1. **Light Alert**: Flash the built-in LED on GP25 if the light level exceeds 3000.
2. **Dark Calibration**: Map the LDR reading to a percentage (0% to 100%) and print the percentage value to the Serial Monitor.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Reading stays stuck at 0 or 4095 | Missing voltage divider | Ensure the 10k fixed resistor is wired correctly between GP26 and GND to pull the voltage down. |

## Mode Notes
This basic analog project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [34 - Pico LDR Dark Detector](34-pico-ldr-dark-detector.md)
- [49 - Pico LDR Dimmer](49-pico-ldr-dimmer.md)
