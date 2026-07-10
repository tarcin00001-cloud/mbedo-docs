# 39 - Pico Rain Sensor

Detect rain conditions and trigger alarms using a rain sensor board.

## Goal
Learn how to interface threshold-based liquid detectors with digital inputs on the Raspberry Pi Pico.

## What You Will Build
A rain notification system:
- **Rain Sensor (GP16)**: Reads `LOW` when water drops land on the sensor plate (active-LOW trigger).
- **Warning LED (GP15)**: Turns ON (Red warning) immediately when rain is detected.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Rain Sensor Board | `button` | Yes (represented by switch button) | Yes (or rain detector module) |
| LED (Red) | `led` | Yes | Yes |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Rain Sensor | VCC | 3.3V | Power supply |
| Rain Sensor | DO (Digital Out) | GP16 | Rain trigger signal |
| Rain Sensor | GND | GND | Ground reference |
| Red LED | Anode | GP15 | Safety warning indicator |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int RAIN_PIN = 16;
const int LED_PIN  = 15;

void setup() {
  pinMode(RAIN_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int rainDetected = digitalRead(RAIN_PIN);

  // Active-LOW logic: comparator pulls DO pin to GND when wet
  if (rainDetected == LOW) {
    digitalWrite(LED_PIN, HIGH); // Rain detected! LED ON
  } else {
    digitalWrite(LED_PIN, LOW);  // Dry, LED OFF
  }

  delay(100);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Rain Sensor** (represented by switch button), and **Red LED** onto the canvas.
2. Connect Rain Sensor: **VCC** to **3.3V**, **DO** to **GP16**, **GND** to **GND**.
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Simulate rain by closing the switch contact representing the wet sensor on canvas.

## Expected Output

Terminal:
```
Simulation active. Rain alarm monitor active.
```

## Expected Canvas Behavior
| Rain Sensor Plate | GP16 Input State | LED state (GP15) | Weather Status |
| --- | --- | --- | --- |
| Dry | HIGH | LOW | Clear |
| Wet (Raining) | LOW | HIGH | **RAIN DETECTED** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `rainDetected == LOW` | Checks if the rain module's comparator has pulled the digital pin LOW due to moisture on the plate. |

## Hardware & Safety Concept: Nickel-plated Sensor Traces
Rain sensor plates consist of interleaved copper traces covered with nickel plating. Water drops landing on the plate bridge the gaps between traces, decreasing the resistance between power and sense lines. The sensor module comparator checks this resistance against a reference potentiometer and outputs a clean digital `LOW` signal during rain events.

## Try This! (Challenges)
1. **Wiper Simulation**: Connect a Servo Motor to GP14. If rain is detected, sweep the servo back and forth to simulate a windshield wiper.
2. **Audio Siren**: Connect a buzzer on GP10 and sound a warning tone when rain starts.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays ON even when dry | Sensitivity potentiometer too high | Turn the potentiometer on the physical comparator board counter-clockwise to reduce sensitivity. |

## Mode Notes
This basic GPIO project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [24 - Pico Limit Switch](24-pico-limit-switch.md)
- [41 - Pico Water Level](41-pico-water-level.md)
