# 25 - Pico Reed Switch

Detect magnetic fields using a mechanical reed switch sensor.

## Goal
Learn how to interface magnetic reed switches with digital inputs on the Raspberry Pi Pico for contact-free position sensing.

## What You Will Build
A magnetic door-open alarm:
- **Reed Switch (GP16)**: Configured with an internal pull-up resistor (active-LOW).
- **Warning LED (GP15)**: Turns ON when a magnet is placed near the reed switch (bringing GP16 LOW).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Reed Switch Module | `button` | Yes (or switch module) | Yes |
| LED (Red) | `led` | Yes | Yes |
| Neodymium Magnet | `magnetic_target` | Yes | Yes |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Reed Switch | Terminal 1 | GP16 | Digital sense pin |
| Reed Switch | Terminal 2 | GND | Ground return |
| Red LED | Anode | GP15 | Magnetic alert indicator |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int REED_PIN = 16;
const int LED_PIN  = 15;

void setup() {
  pinMode(REED_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int magnetDetected = digitalRead(REED_PIN);

  // Active-LOW check: switch closes (LOW) when magnet is close
  if (magnetDetected == LOW) {
    digitalWrite(LED_PIN, HIGH); // Magnet present, alert ON
  } else {
    digitalWrite(LED_PIN, LOW);  // No magnet, alert OFF
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Reed Switch** (represented by switch button), and **Red LED** onto the canvas.
2. Connect Reed Switch: **Terminal 1** to **GP16**, **Terminal 2** to **GND**.
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Simulate magnet approach by closing the switch contact on canvas.

## Expected Output

Terminal:
```
Simulation active. Magnetic field monitor online.
```

## Expected Canvas Behavior
| Magnet Position | Reed Switch State | GP16 Voltage | LED state (GP15) |
| --- | --- | --- | --- |
| Far Away | Open | 3.3V (HIGH) | LOW (OFF) |
| Close By | Closed | 0V (LOW) | HIGH (ON) |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `magnetDetected == LOW` | Verifies if the magnetic contacts have snapped together under a magnetic field, bringing the pin to ground. |

## Hardware & Safety Concept: Reed Switch Architecture
A reed switch consists of two overlapping, flexible metal reeds sealed in a glass tube filled with inert gas. When a magnetic field approaches, the reeds magnetize with opposite polarities and pull together, closing the circuit. Because the contacts are sealed, they are immune to dust, moisture, and corrosion, making them ideal for home security door/window sensors.

## Try This! (Challenges)
1. **Window Guard**: Add a buzzer on GP14 and sound a rapid pulse alarm if the door/window is opened (meaning the magnet moves *away* from the reed switch).
2. **Contact Counter**: Log to the Serial Monitor every time the reed switch opens or closes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED glows dimly or flashes | Loose connection | Check breadboard connections; mechanical reed switches require solid electrical contacts. |

## Mode Notes
This basic GPIO project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [17 - Pico Button Pull-up](17-pico-button-pullup.md)
- [24 - Pico Limit Switch](24-pico-limit-switch.md)
