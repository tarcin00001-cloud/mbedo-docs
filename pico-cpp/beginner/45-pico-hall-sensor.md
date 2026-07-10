# 45 - Pico Hall Sensor

Detect magnetic fields using a digital Hall Effect sensor.

## Goal
Learn how to interface solid-state magnetic sensors with digital inputs on the Raspberry Pi Pico.

## What You Will Build
A contact-free proximity checker:
- **Hall Effect Sensor (GP16)**: Reads `LOW` when the South pole of a magnet is close (active-LOW trigger).
- **Warning LED (GP15)**: Turns ON (Red alert) immediately when a magnet is detected.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Hall Effect Sensor Module | `button` | Yes (represented by switch button) | Yes (or Hall module) |
| LED (Red) | `led` | Yes | Yes |
| Magnet | `magnetic_target` | Yes | Yes |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Hall Sensor | VCC | 3.3V | Power supply |
| Hall Sensor | OUT | GP16 | Sensor output |
| Hall Sensor | GND | GND | Ground reference |
| Red LED | Anode | GP15 | Proximity indicator |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int HALL_PIN = 16;
const int LED_PIN  = 15;

void setup() {
  pinMode(HALL_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int magnetDetected = digitalRead(HALL_PIN);

  // Active-LOW logic: Hall sensor pulls OUT to GND on magnetic contact
  if (magnetDetected == LOW) {
    digitalWrite(LED_PIN, HIGH); // Magnet near, LED ON
  } else {
    digitalWrite(LED_PIN, LOW);  // Magnet far, LED OFF
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Hall Effect Sensor** (represented by switch button), and **Red LED** onto the canvas.
2. Connect Hall Sensor: **VCC** to **3.3V**, **OUT** to **GP16**, **GND** to **GND**.
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Simulate magnet contact by closing the switch contact on canvas.

## Expected Output

Terminal:
```
Simulation active. Magnetic Hall sensor monitor online.
```

## Expected Canvas Behavior
| Magnet Position | GP16 Input State | LED state (GP15) | Proximity status |
| --- | --- | --- | --- |
| Magnet Far | HIGH | LOW | Open |
| Magnet Near | LOW | HIGH | **MAGNET CLOSE** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `magnetDetected == LOW` | Checks if the semiconductor Hall element has detected a magnetic flux density exceeding the trigger threshold. |

## Hardware & Safety Concept: Hall Effect Principles
When a current flows through a semiconductor placed in a magnetic field, the field exerts a transverse force (Lorentz force) on the charge carriers, pushing them to one side. This generates a small measurable voltage difference across the semiconductor sheet (the Hall voltage). A built-in comparator checks this voltage and outputs a clean digital `LOW` signal when a magnetic pole is present.

## Try This! (Challenges)
1. **Passage Counter**: Print a message to the Serial Monitor every time the magnet passes the sensor (e.g. for counting wheel rotations on a bicycle).
2. **Double Alert**: Add a buzzer on GP14 and sound a brief tone on magnetic detection.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sensor doesn't trigger with magnet | Wrong magnet pole | Hall effect sensors are polar. If it doesn't trigger, flip the magnet over to present the opposite pole (South) to the sensor face. |

## Mode Notes
This basic GPIO project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [25 - Pico Reed Switch](25-pico-reed-switch.md)
- [37 - Pico IR Obstacle](37-pico-ir-obstacle.md)
