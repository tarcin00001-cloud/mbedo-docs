# 44 - Pico Flame Sensor

Detect open flames and trigger safety alarms using a Flame Sensor.

## Goal
Learn how to interface optical fire sensors with digital input pins on the Raspberry Pi Pico to detect flame infrared signatures.

## What You Will Build
A fire alarm node:
- **Flame Sensor (GP16)**: Reads `LOW` when a fire is detected near the sensor (active-LOW trigger).
- **Warning LED (GP15)**: Turns ON (Red alert) immediately when a flame is detected.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Flame Sensor Module | `button` | Yes (represented by switch button) | Yes (or IR flame module) |
| LED (Red) | `led` | Yes | Yes |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Flame Sensor | VCC | 3.3V | Power supply |
| Flame Sensor | DO (Digital Out) | GP16 | Flame trigger signal |
| Flame Sensor | GND | GND | Ground reference |
| Red LED | Anode | GP15 | Fire warning beacon |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int FLAME_PIN = 16;
const int LED_PIN   = 15;

void setup() {
  pinMode(FLAME_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int fireDetected = digitalRead(FLAME_PIN);

  // Active-LOW logic: comparator pulls DO pin to GND when IR signature matches
  if (fireDetected == LOW) {
    digitalWrite(LED_PIN, HIGH); // FIRE! Alert ON
  } else {
    digitalWrite(LED_PIN, LOW);  // Safe, Alert OFF
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Flame Sensor** (represented by switch button), and **Red LED** onto the canvas.
2. Connect Flame Sensor: **VCC** to **3.3V**, **DO** to **GP16**, **GND** to **GND**.
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Simulate a fire by closing the switch contact representing flame detection on canvas.

## Expected Output

Terminal:
```
Simulation active. Fire alarm guard online.
```

## Expected Canvas Behavior
| Flame Presence | GP16 Input State | LED state (GP15) | Safety Status |
| --- | --- | --- | --- |
| No Fire | HIGH | LOW | Safe |
| Fire Detected | LOW | HIGH | **FIRE ALARM ACTIVE** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `fireDetected == LOW` | Verifies if the flame sensor phototransistor has conducted, pulling the comparator output to ground. |

## Hardware & Safety Concept: Infrared Flame Signatures
Flame sensor modules utilize a specialized phototransistor sensitive to infrared radiation in the wavelength range of 760nm to 1100nm. This matches the emission spectrum of hot carbon dioxide ($CO_2$) gas generated in open fires. By filtering out visible light, the sensor focuses on infrared wavelengths, though bright sunlight or incandescent bulbs can still trigger false alerts.

## Try This! (Challenges)
1. **Siren Alert**: Connect a buzzer on GP14 and sound a rapid warning tone if a fire is detected.
2. **Auto-Extinguisher**: Connect a relay on GP10 (driving a water valve/pump) and activate it when a flame is detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers near home lighting | Lightbulb interference | Incandescent and halogen bulbs emit large amounts of infrared light. Test the sensor away from direct bulbs. |

## Mode Notes
This basic GPIO project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [24 - Pico Limit Switch](24-pico-limit-switch.md)
- [39 - Pico Rain Sensor](39-pico-rain-sensor.md)
