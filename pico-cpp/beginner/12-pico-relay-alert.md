# 12 - Pico Relay Alert

Synchronize a relay module state with a flashing warning LED to build a visible safety alert indicator.

## Goal
Learn how to coordinate multiple digital outputs (relay and warning LED) to trigger simultaneously during alarm phases.

## What You Will Build
A synchronized warning beacon:
- **Active Warning (1s)**: Relay is activated (IN = HIGH) and Warning LED (GP15) is turned ON.
- **Idle Warning (1s)**: Relay is deactivated (IN = LOW) and Warning LED is turned OFF.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 220-ohm Resistor | `resistor` | Optional | Yes (pull-down) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Relay Module | VCC | 5V | Power supply |
| Relay Module | IN | GP10 | Relay control signal |
| Relay Module | GND | GND | Ground reference |
| Red LED | Anode | GP15 | LED warning pin |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int RELAY_PIN = 10;
const int LED_PIN   = 15;

void setup() {
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  // Alert Phase
  digitalWrite(RELAY_PIN, HIGH);
  digitalWrite(LED_PIN, HIGH);
  delay(1000);

  // Idle Phase
  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(LED_PIN, LOW);
  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Relay Module**, and **Red LED** onto the canvas.
2. Connect Relay: **VCC** to **5V**, **IN** to **GP10**, **GND** to **GND**.
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Observe the relay clicking and LED flashing in perfect sync.

## Expected Output

Terminal:
```
Simulation active. GP10 (Relay) and GP15 (LED) synchronized warning.
```

## Expected Canvas Behavior
| Cycle Time | GP10 (Relay) | GP15 (LED) | Alert State |
| --- | --- | --- | --- |
| 0–1.0 s | HIGH | HIGH | Active (Light + Click) |
| 1.0–2.0 s | LOW | LOW | Inactive (Dark + Silent) |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(LED_PIN, HIGH)` | Powers the LED anode simultaneously as the relay coil is driven to HIGH. |

## Hardware & Safety Concept: Optical Isolation
Many relay boards have built-in **optocouplers** (optical isolators). Inside the optocoupler, the microcontroller signal lights a tiny internal infrared LED, which activates a light-sensitive phototransistor to drive the relay coil circuit. This ensures complete optical separation between the microcontroller and the relay coil electromotive forces.

## Try This! (Challenges)
1. **Asynchronous Warning**: Program the LED to flash twice rapidly (100 ms) while the relay stays closed, then open the relay and wait 2 seconds.
2. **Audio Sync**: Add a buzzer on GP14 and configure it to sound only when the relay and LED are active.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED flashes but relay doesn't click | Missing power on VCC | Check that the relay module VCC pin is connected to the Pico 5V pin, not a lower logic pin. |

## Mode Notes
This multi-device digital output project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [11 - Pico Relay Toggle](11-pico-relay-toggle.md)
- [13 - Pico DC Fan Switch](13-pico-dc-fan-switch.md)
