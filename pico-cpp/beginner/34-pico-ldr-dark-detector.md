# 34 - Pico LDR Dark Detector

Build an automatic night light that turns ON an LED when ambient light levels fall below a threshold.

## Goal
Learn how to use threshold logic with analog inputs to trigger digital outputs based on environmental conditions.

## What You Will Build
An automatic night light:
- **LDR Sensor (GP26)**: Monitors ambient light.
- **Warning LED (GP15)**: Automatically turns ON when the room goes dark (LDR < 1000) and turns OFF in bright light.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LDR (Photoresistor) | `photoresistor` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 10k-ohm Resistor | `resistor` | Optional | Yes (voltage divider) |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| LDR | Pin 1 | 3.3V | Power supply |
| LDR | Pin 2 | GP26 | Analog input (wire 10k resistor from GP26 to GND) |
| LDR | Pin 2 | GND | Ground return via resistor |
| Red LED | Anode | GP15 | Light control |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int LDR_PIN = 26;
const int LED_PIN = 15;

const int DARK_THRESHOLD = 1000; // Threshold for dark detection

void setup() {
  pinMode(LDR_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int lightLevel = analogRead(LDR_PIN);

  // If light drops below threshold, turn night light ON
  if (lightLevel < DARK_THRESHOLD) {
    digitalWrite(LED_PIN, HIGH);
  } else {
    digitalWrite(LED_PIN, LOW);
  }

  delay(100);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **LDR Sensor**, and **Red LED** onto the canvas.
2. Connect LDR: **Pin 1** to **3V3**, **Pin 2** to **GP26** (with 10k resistor to GND).
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Drag the LDR light slider on canvas to dark (minimum) and watch the LED turn ON.

## Expected Output

Terminal:
```
Simulation active. Dark detector active. Threshold: 1000.
```

## Expected Canvas Behavior
| Light Slider level | GP26 Input Value | LED state (GP15) | Light Status |
| --- | --- | --- | --- |
| Bright (100%) | 3500 | LOW | OFF (Daylight) |
| Medium (50%) | 2200 | LOW | OFF (Room light) |
| Dark (0%) | 400 | HIGH | **ON (Night Light Active)** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `lightLevel < DARK_THRESHOLD` | Boolean check to see if the photoresistor output voltage drops below the 1000 threshold. |

## Hardware & Safety Concept: Sensor Hysteresis
In real-world street lighting systems, simple thresholds can cause the light to flicker rapidly at dusk when light levels hover precisely near the threshold. To prevent this, engineers use **hysteresis** in software (e.g. turn ON when light < 800, but only turn OFF when light > 1200), ensuring a clean, stable transition.

## Try This! (Challenges)
1. **Adding Alert Audio**: Connect a buzzer on GP14 and sound a brief chirp when the night light turns ON.
2. **Reverse Solar Tracker**: Rewrite the code so the LED turns ON only during bright sunlight, acting as a solar charger status indicator.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays ON constantly | Threshold too high | Adjust `DARK_THRESHOLD` in the code to a lower value (e.g. 800) to match your environment. |

## Mode Notes
This basic threshold logic project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [33 - Pico LDR Sensor](33-pico-ldr-sensor.md)
- [49 - Pico LDR Dimmer](49-pico-ldr-dimmer.md)
