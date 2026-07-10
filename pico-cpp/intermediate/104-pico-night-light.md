# 104 - Pico Night Light

Build an energy-saving night light that closes a relay switch only when it is dark and motion is detected.

## Goal
Learn how to combine analog ambient light sensors (LDR) and digital motion sensors (PIR) to build smart control logic for lighting relays.

## What You Will Build
An automatic night light controller:
- **LDR Photoresistor (GP26)**: Measures ambient room brightness.
- **PIR Motion Sensor (GP16)**: Detects human movement.
- **Relay Module (GP10)**: Closes contact (light ON) only if it is dark AND motion is detected.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LDR Photoresistor | `ldr` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| 10k-ohm Resistor | `resistor` | Optional | Yes (voltage divider) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| LDR | Pin 1 | 3.3V | Voltage divider high |
| LDR | Pin 2 (Signal) | GP26 | Analog input (requires 10k pull-down to GND) |
| PIR Sensor | VCC | 5V | Power supply |
| PIR Sensor | OUT | GP16 | Motion sense pin |
| PIR Sensor | GND | GND | Ground return |
| Relay Module | VCC | 5V | Coil power |
| Relay Module | IN | GP10 | Light switch line |
| Relay Module | GND | GND | Ground return |

## Code
```cpp
const int LDR_PIN   = 26;
const int PIR_PIN   = 16;
const int RELAY_PIN = 10;

// Dark threshold: low values indicate darkness
const int DARK_LIMIT = 1000;

void setup() {
  pinMode(LDR_PIN, INPUT);
  pinMode(PIR_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  
  digitalWrite(RELAY_PIN, LOW); // Start with light OFF
}

void loop() {
  int lightLevel = analogRead(LDR_PIN);
  int motion = digitalRead(PIR_PIN);

  // Logical condition: Dark AND Motion detected
  if (lightLevel < DARK_LIMIT && motion == HIGH) {
    digitalWrite(RELAY_PIN, HIGH); // Turn light ON
    delay(5000);                   // Keep active for 5 seconds
  } else {
    digitalWrite(RELAY_PIN, LOW);  // Turn light OFF
  }

  delay(200);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **LDR**, **PIR Sensor**, and **Relay Module** onto the canvas.
2. Connect LDR to **GP26** (with 10k pull-down to GND), PIR OUT to **GP16**, and Relay IN to **GP10**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Set the LDR slider to dark (low values) and trigger the PIR sensor to activate the relay.

## Expected Output

Terminal:
```
Simulation active. Smart night light controller active.
```

## Expected Canvas Behavior
| Light Slider level | Motion (PIR) state | GP10 (Relay state) | Status |
| --- | --- | --- | --- |
| Bright (> 1500) | HIGH | LOW | OFF (Too bright) |
| Dark (< 1000) | LOW | LOW | OFF (No movement) |
| Dark (< 1000) | HIGH | HIGH | **ON (Light activated)** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `lightLevel < DARK_LIMIT && motion == HIGH` | Evaluates both criteria before closing the relay contacts, preventing the light from turning ON during the day. |

## Hardware & Safety Concept: Energy Saving Lighting Controls
Using a dual-sensor condition (darkness AND motion) ensures lights only turn ON when needed. This approach is widely used in smart building designs, reducing energy consumption in hallways, stairs, and parking garages by up to 80% compared to systems that remain ON continuously overnight.

## Try This! (Challenges)
1. **Auditory Indicator**: Connect a buzzer on GP14 and sound a brief confirmation chirp when the light turns ON.
2. **Dynamic Alarm**: Flash a warning LED on GP15 if motion is detected in the dark for more than 15 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Light turns on during daytime | LDR threshold too high | Check the raw analog readings from the LDR during the day and adjust `DARK_LIMIT` to a lower value. |

## Mode Notes
This multi-device analog control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [34 - Pico LDR Sensor](../../beginner/34-pico-ldr-sensor.md)
- [36 - Pico PIR Motion](../../beginner/36-pico-pir-motion.md)
- [103 - Pico Smart Doorbell](103-pico-smart-doorbell.md)
