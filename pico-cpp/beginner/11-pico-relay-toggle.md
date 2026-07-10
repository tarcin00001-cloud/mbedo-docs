# 11 - Pico Relay Toggle

Toggle a relay module state using digital outputs.

## Goal
Learn how to use digital output pins on the Raspberry Pi Pico to actuate mechanical switches for controlling high-power loads.

## What You Will Build
A relay clicking simulator:
- **Relay Module (GP10)**: Alternates between active (Normally Open closed) and inactive states every 1.5 seconds, accompanied by a simulated mechanical click.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Relay Module | VCC | 5V (or 3.3V depending on module) | Power supply |
| Relay Module | IN | GP10 | Control signal |
| Relay Module | GND | GND | Ground reference |

## Code
```cpp
const int RELAY_PIN = 10;

void setup() {
  pinMode(RELAY_PIN, OUTPUT);
}

void loop() {
  digitalWrite(RELAY_PIN, HIGH); // Activate relay (click)
  delay(1500);
  digitalWrite(RELAY_PIN, LOW);  // Deactivate relay (release)
  delay(1500);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Relay Module** onto the canvas.
2. Connect Relay pins: **VCC** to **3VSYS** (or 5V), **IN** to **GP10**, and **GND** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Listen to the simulated click and observe the relay switch state change.

## Expected Output

Terminal:
```
Simulation active. GP10 (Relay) cycling ON/OFF.
```

## Expected Canvas Behavior
| Time (ms) | Relay Pin (GP10) | Relay state |
| --- | --- | --- |
| 0–1500 | HIGH | Active (NO closed, NC open) |
| 1500–3000 | LOW | Inactive (NO open, NC closed) |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(RELAY_PIN, HIGH)` | Drives GP10 HIGH, energizing the internal optocoupler/transistor that powers the relay coil. |

## Hardware & Safety Concept: Galvanic Isolation
Relays provide **galvanic isolation** between low-voltage microcontrollers (3.3V/5V) and high-voltage loads (like 110V/220V AC motors or lights). The electrical signal from the microcontroller powers an electromagnet (coil) that mechanically pulls a switch arm, ensuring that high-voltage current cannot flow back to damage the Pico.

## Try This! (Challenges)
1. **Pulsing Duty Cycle**: Modify the delays to keep the relay closed (ON) for 5 seconds and open (OFF) for 1 second.
2. **Dual Relay Sequence**: Add a second relay on GP11 and configure them to toggle alternately.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay does not click | Insufficient drive voltage | Some relays require 5V to actuate the electromagnet coil. Try connecting the relay VCC to the Pico's VBUS (5V USB pin) instead of 3.3V. |

## Mode Notes
This basic relay control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [12 - Pico Relay Alert](12-pico-relay-alert.md)
- [13 - Pico DC Fan Switch](13-pico-dc-fan-switch.md)
