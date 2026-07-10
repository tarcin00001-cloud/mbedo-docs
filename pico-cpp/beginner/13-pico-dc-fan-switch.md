# 13 - Pico DC Fan Switch

Control a 5V DC cooling fan on/off cycle using a digital switch pin.

## Goal
Learn how to use digital output pins on the Raspberry Pi Pico to control mechanical motors via transistor or relay switches.

## What You Will Build
A periodic cooling cycle:
- **DC Fan Control (GP10)**: Activates the fan for 4 seconds, followed by 4 seconds of idle cooldown.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DC Motor (Fan) | `dc_motor` | Yes | Yes |
| Relay Module (or NPN Transistor)| `relay` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Relay Module | VCC | 5V | Power supply |
| Relay Module | IN | GP10 | Fan trigger pin |
| Relay Module | GND | GND | Ground reference |
| DC Motor | Positive | Relay NO | Motor gets 5V when relay is closed |
| DC Motor | Negative | GND | Ground return |

## Code
```cpp
const int FAN_PIN = 10;

void setup() {
  pinMode(FAN_PIN, OUTPUT);
}

void loop() {
  digitalWrite(FAN_PIN, HIGH); // Turn fan ON
  delay(4000);
  digitalWrite(FAN_PIN, LOW);  // Turn fan OFF
  delay(4000);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Relay Module**, and **DC Motor** onto the canvas.
2. Connect Relay: **VCC** to **5V**, **IN** to **GP10**, **GND** to **GND**.
3. Connect DC Motor: **Positive** to Relay **NO** (Normally Open terminal), and **Negative** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Observe the DC motor spinning on and off in 4-second intervals.

## Expected Output

Terminal:
```
Simulation active. GP10 (Fan relay) cycling.
```

## Expected Canvas Behavior
| Time (ms) | Fan Pin (GP10) | Motor Spin State |
| --- | --- | --- |
| 0–4000 | HIGH | Spinning |
| 4000–8000 | LOW | Stopped |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(FAN_PIN, HIGH)` | Drives GP10 HIGH, switching on the driver relay to spin the DC motor. |

## Hardware & Safety Concept: Flyback Diodes
DC motors contain coils that act as inductors. When current is suddenly cut off (by deactivating a transistor or relay), the magnetic field collapses, generating a high-voltage spike in the reverse direction. On real hardware, a **flyback diode** must be placed in parallel with the motor to route this spike safely, protecting the control circuitry from burning out.

## Try This! (Challenges)
1. **Temperature Simulation**: Add an LDR (simulating temperature sensor) on GP26. If the analog reading is high, turn the fan ON, otherwise turn it OFF.
2. **Speed Indicator**: Wire a blue LED to GP15 that turns ON only when the fan is active.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Fan turns ON when code is LOW | Wired to NC instead of NO | Confirm that the motor wire connects to the Normally Open (NO) relay terminal, not the Normally Closed (NC) terminal. |

## Mode Notes
This basic motor control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [11 - Pico Relay Toggle](11-pico-relay-toggle.md)
- [12 - Pico Relay Alert](12-pico-relay-alert.md)
