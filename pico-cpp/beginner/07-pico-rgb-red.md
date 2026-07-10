# 07 - Pico RGB Red

Light up a common-cathode RGB LED in static Red.

## Goal
Learn how to wire an RGB LED and control individual primary color channels using digital outputs on the Raspberry Pi Pico.

## What You Will Build
A red light indicator:
- **RGB LED Red Channel (GP11)**: Driven HIGH to output Red.
- **Green Channel (GP12) & Blue Channel (GP13)**: Driven LOW to remain OFF.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| RGB LED (Common Cathode)| `led` | Yes | Yes (or standard tri-color) |
| 220-ohm Resistor | `resistor` | Optional | Yes (pull-down) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| RGB LED | Red Pin | GP11 | Control Red channel |
| RGB LED | Common Cathode | GND | Shared ground return |
| RGB LED | Green Pin | GP12 | Control Green channel |
| RGB LED | Blue Pin | GP13 | Control Blue channel |

## Code
```cpp
const int RED_PIN = 11;
const int GRN_PIN = 12;
const int BLU_PIN = 13;

void setup() {
  pinMode(RED_PIN, OUTPUT);
  pinMode(GRN_PIN, OUTPUT);
  pinMode(BLU_PIN, OUTPUT);

  // Set to Static Red
  digitalWrite(RED_PIN, HIGH);
  digitalWrite(GRN_PIN, LOW);
  digitalWrite(BLU_PIN, LOW);
}

void loop() {
  // Static state - nothing to update in loop
  delay(100);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **RGB LED** onto the canvas.
2. Connect RGB LED pins: **Red** to **GP11**, **Common Cathode** (GND) to **GND**, **Green** to **GP12**, **Blue** to **GP13**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Observe the RGB LED shine bright Red.

## Expected Output

Terminal:
```
Simulation active. RGB LED set to static Red.
```

## Expected Canvas Behavior
| GP11 (Red Pin) | GP12 (Green Pin) | GP13 (Blue Pin) | RGB LED Color |
| --- | --- | --- | --- |
| HIGH | LOW | LOW | Red |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(RED_PIN, HIGH)` | Drives GP11 to 3.3V, lighting up the internal red LED chip. |

## Hardware & Safety Concept: Common Cathode vs. Common Anode
RGB LEDs contain three independent LED dies (Red, Green, Blue) sharing a common pin.
- **Common Cathode (CC)**: The shared pin is GND. You write `HIGH` to light a channel.
- **Common Anode (CA)**: The shared pin is VCC. You write `LOW` to light a channel.
Using the wrong logic will invert your color outputs. This project uses Common Cathode logic.

## Try This! (Challenges)
1. **Yellow Mix**: Write both `RED_PIN` and `GRN_PIN` to `HIGH` to mix Red and Green light, producing Yellow.
2. **Setup to Loop**: Move the write commands into the `loop()` and toggle Red ON and OFF every 1 second.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED glows Blue instead of Red | Swapped pin wiring | Confirm that GP11 is connected to the Red pin and GP13 to the Blue pin. |

## Mode Notes
This basic multi-channel output project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [08 - Pico RGB Green](08-pico-rgb-green.md)
- [10 - Pico RGB Cycle](10-pico-rgb-cycle.md)
