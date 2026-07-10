# 09 - Pico RGB Blue

Light up a common-cathode RGB LED in static Blue.

## Goal
Learn how to control individual primary color channels of an RGB LED on the Raspberry Pi Pico.

## What You Will Build
A blue light indicator:
- **RGB LED Blue Channel (GP13)**: Driven HIGH to output Blue.
- **Red Channel (GP11) & Green Channel (GP12)**: Driven LOW to remain OFF.

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

  // Set to Static Blue
  digitalWrite(RED_PIN, LOW);
  digitalWrite(GRN_PIN, LOW);
  digitalWrite(BLU_PIN, HIGH);
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
4. Observe the RGB LED shine bright Blue.

## Expected Output

Terminal:
```
Simulation active. RGB LED set to static Blue.
```

## Expected Canvas Behavior
| GP11 (Red Pin) | GP12 (Green Pin) | GP13 (Blue Pin) | RGB LED Color |
| --- | --- | --- | --- |
| LOW | LOW | HIGH | Blue |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(BLU_PIN, HIGH)` | Drives GP13 to 3.3V, lighting up the internal blue LED chip. |

## Hardware & Safety Concept: Mixing Colors
Driving multiple LED channels simultaneously blends primary wavelengths of light (Red, Green, Blue) together inside the plastic housing, allowing you to generate secondary colors. Combining all three at equal intensity creates white light.

## Try This! (Challenges)
1. **Magenta Mix**: Write both `RED_PIN` and `BLU_PIN` to `HIGH` to mix Red and Blue light, producing Magenta.
2. **Flashing Blue**: Move the write commands into the `loop()` and toggle Blue ON and OFF every 500 ms.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Color appears dim | Output pin current limit | Avoid running too many LEDs directly from GPIO pins without external driver circuits or buffer resistors. |

## Mode Notes
This basic multi-channel output project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [07 - Pico RGB Red](07-pico-rgb-red.md)
- [08 - Pico RGB Green](08-pico-rgb-green.md)
