# 108 - Pico Intrusion Alarm

Build an invisible laser tripwire alarm that activates a relay if the light beam is crossed.

## Goal
Learn how to control light transmitters (lasers) and read analog light receivers (LDRs) to build tripwire security circuits.

## What You Will Build
A laser tripwire barrier:
- **Laser Diode (GP16)**: Kept constantly ON.
- **LDR Photoresistor (GP26)**: Aligned to receive the laser beam.
- **Relay Module (GP10)**: Closes contact (triggering an alarm) if the beam is broken.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Laser Diode Module | `led` | Yes (represented by LED) | Yes (low-power laser pointer) |
| LDR Photoresistor | `ldr` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| 10k-ohm Resistor | `resistor` | Optional | Yes (voltage divider) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Laser Diode | Anode | GP16 | Laser power control |
| Laser Diode | Cathode | GND | Ground return |
| LDR | Pin 1 | 3.3V | Voltage divider high |
| LDR | Pin 2 (Signal) | GP26 | Analog input (requires 10k pull-down to GND) |
| Relay Module | VCC | 5V | Coil power |
| Relay Module | IN | GP10 | Siren switch line |
| Relay Module | GND | GND | Ground return |

## Code
```cpp
const int LASER_PIN  = 16;
const int LDR_PIN    = 26;
const int RELAY_PIN  = 10;

// Tripwire limit: light level drops below threshold when laser beam is blocked.
// Laser beam directly hitting LDR reads >3000. Blocked beam reads <1500.
const int TRIP_THRESHOLD = 2000;

void setup() {
  pinMode(LASER_PIN, OUTPUT);
  pinMode(LDR_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);

  // Turn ON laser diode
  digitalWrite(LASER_PIN, HIGH);
  
  // Start with alarm OFF
  digitalWrite(RELAY_PIN, LOW);
}

void loop() {
  int beamValue = analogRead(LDR_PIN);

  // If laser beam is broken (value drops below threshold)
  if (beamValue < TRIP_THRESHOLD) {
    digitalWrite(RELAY_PIN, HIGH); // Alarm ON
    delay(2000);                   // Hold alarm active for 2 seconds
  } else {
    digitalWrite(RELAY_PIN, LOW);  // Alarm OFF
  }

  delay(50); // Fast scan rate
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Laser Diode** (represented by LED), **LDR**, and **Relay Module** onto the canvas.
2. Connect Laser to **GP16**, LDR to **GP26** (with 10k pull-down), and Relay IN to **GP10**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the LDR slider down to simulate breaking the laser beam and check that the relay clicks ON.

## Expected Output

Terminal:
```
Simulation active. Laser tripwire security loop active.
```

## Expected Canvas Behavior
| LDR Light level slider | GP10 (Relay state) | Status |
| --- | --- | --- |
| Bright (> 2500) | LOW | Beam Intact (Safe) |
| Dark (< 1500) | HIGH (Closed) | **TRIPWIRE BREACHED ALARM ON** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(LASER_PIN, HIGH)` | Powers the laser diode to shine light onto the LDR sensor. |

## Hardware & Safety Concept: Laser Alignment and Eye Safety
When building laser tripwires, alignment is critical. The laser beam must hit the small surface of the LDR precisely. A small plastic tube placed around the LDR (a light shield) helps filter out ambient room light, making the sensor react only to the laser. **Always use Class 1 or Class 2 low-power lasers (under 1mW)**, and never look directly into the beam to avoid eye injury.

## Try This! (Challenges)
1. **Latching Alarm**: Modify the code so that if the beam is broken, the relay stays active until the Pico is reset.
2. **Audio alarm**: Connect a buzzer on GP14 and play warning beeps when the tripwire is breached.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly | Ambient light interference | Verify the laser beam is landing directly on the LDR. Use a plastic tube shield to block surrounding room light from hitting the sensor. |

## Mode Notes
This multi-device analog control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [11 - Pico Relay Toggle](../../beginner/11-pico-relay-toggle.md)
- [34 - Pico LDR Sensor](../../beginner/34-pico-ldr-sensor.md)
- [104 - Pico Night Light](104-pico-night-light.md)
