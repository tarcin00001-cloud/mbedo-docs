# 30 - Pico Laser Receiver

Detect laser beams and trigger alerts using a photo-receiver module.

## Goal
Learn how to interface optical sensor modules with digital inputs on the Raspberry Pi Pico to detect laser beam interruptions.

## What You Will Build
A laser tripwire warning indicator:
- **Laser Receiver (GP16)**: Reads `HIGH` when the laser beam hits the sensor plate, and `LOW` when the beam is broken (interrupted).
- **Warning LED (GP15)**: Turns ON (Red warning) when the beam is broken (interrupted), and remains OFF during normal aligned operation.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Laser Receiver Module | `button` | Yes (represented by switch button) | Yes (or photoresistor module) |
| Laser Transmitter Module | `laser` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Laser Receiver | VCC | 3.3V | Power supply |
| Laser Receiver | OUT | GP16 | Digital signal output |
| Laser Receiver | GND | GND | Ground reference |
| Red LED | Anode | GP15 | Safety warning indicator |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int LASER_IN  = 16;
const int ALARM_LED = 15;

void setup() {
  pinMode(LASER_IN, INPUT);
  pinMode(ALARM_LED, OUTPUT);
}

void loop() {
  int laserState = digitalRead(LASER_IN);

  // Laser aligned: GP16 is HIGH. Beam broken: GP16 goes LOW.
  if (laserState == LOW) {
    digitalWrite(ALARM_LED, HIGH); // Tripwire broken! Alarm ON
  } else {
    digitalWrite(ALARM_LED, LOW);  // Aligned, Alarm OFF
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Laser Receiver** (represented by switch button), and **Red LED** onto the canvas.
2. Connect Laser Receiver: **VCC** to **3.3V**, **OUT** to **GP16**, **GND** to **GND**.
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Simulate breaking the beam by opening the switch contact on canvas.

## Expected Output

Terminal:
```
Simulation active. Laser safety tripwire online.
```

## Expected Canvas Behavior
| Beam Alignment | GP16 Input State | LED state (GP15) | Safety Status |
| --- | --- | --- | --- |
| Aligned (Hit) | HIGH | LOW | Normal |
| Interrupted (Broken) | LOW | HIGH | **TRIPPED (Intrusion Alert)** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `laserState == LOW` | Detects when the light intensity falling on the receiver module drops below the detection threshold. |

## Hardware & Safety Concept: Laser Safety Zones
Active laser receiver modules use a photodiode combined with a comparator circuit. The comparator outputs a clean digital `HIGH` when it detects light matching the laser wavelength (usually 650nm red). Unlike raw photoresistors, this digital filtering prevents ambient sunlight or ceiling lights from causing false alignment signals.

## Try This! (Challenges)
1. **Latching Tripwire**: Modify the code so that if the beam is broken even once, the alarm LED stays ON until a reset button (GP17) is pressed.
2. **Audio Siren**: Connect a buzzer to GP14 and sound a rapid pulse siren when the beam is broken.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers in ambient light | Misalignment | Ensure the laser transmitter module is aimed directly at the receiver sensor plate on real hardware. |

## Mode Notes
This basic GPIO project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [16 - Pico Button LED](16-pico-button-led.md)
- [24 - Pico Limit Switch](24-pico-limit-switch.md)
