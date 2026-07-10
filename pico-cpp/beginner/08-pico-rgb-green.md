# 08 - Pico RGB Green

Light up a common-cathode RGB LED in static Green.

## Goal
Learn how to control individual primary color channels of an RGB LED on the Raspberry Pi Pico.

## What You Will Build
A green light indicator:
- **RGB LED Green Channel (GP12)**: Driven HIGH to output Green.
- **Red Channel (GP11) & Blue Channel (GP13)**: Driven LOW to remain OFF.

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

  // Set to Static Green
  digitalWrite(RED_PIN, LOW);
  digitalWrite(GRN_PIN, HIGH);
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
4. Observe the RGB LED shine bright Green.

## Expected Output

Terminal:
```
Simulation active. RGB LED set to static Green.
```

## Expected Canvas Behavior
| GP11 (Red Pin) | GP12 (Green Pin) | GP13 (Blue Pin) | RGB LED Color |
| --- | --- | --- | --- |
| LOW | HIGH | LOW | Green |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(GRN_PIN, HIGH)` | Drives GP12 to 3.3V, lighting up the internal green LED chip. |

## Hardware & Safety Concept: Driving Voltages
Red LEDs require a lower forward voltage (around 1.8V to 2.0V) compared to Green and Blue LEDs (which require 3.0V to 3.2V). When wiring them on breadboards, using the same resistor value (like 220 ohms) across all pins will cause the Red channel to appear brighter than Green or Blue. Adjusting resistor values balances the intensity.

## Try This! (Challenges)
1. **Cyan Mix**: Write both `GRN_PIN` and `BLU_PIN` to `HIGH` to mix Green and Blue light, producing Cyan.
2. **Flashing Green**: Move the write commands into the `loop()` and toggle Green ON and OFF every 500 ms.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED does not light up at all | Cathode wired to a GPIO instead of GND | Verify that the common pin (usually the longest leg) is wired to a physical GND pin. |

## Mode Notes
This basic multi-channel output project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [07 - Pico RGB Red](07-pico-rgb-red.md)
- [09 - Pico RGB Blue](09-pico-rgb-blue.md)
