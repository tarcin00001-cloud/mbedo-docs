# 10 - Pico RGB Cycle

Cycle the RGB LED through primary colors in a continuous loop.

## Goal
Learn how to create chronological shifting patterns across multiple digital outputs to blend and transition light colors.

## What You Will Build
A color-changing indicator:
- **Red Phase (1s)**: Red ON, Green/Blue OFF.
- **Green Phase (1s)**: Green ON, Red/Blue OFF.
- **Blue Phase (1s)**: Blue ON, Red/Green OFF.
- **White Phase (1s)**: Red, Green, and Blue all ON.

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
}

void loop() {
  // 1. Red Phase
  digitalWrite(RED_PIN, HIGH);
  digitalWrite(GRN_PIN, LOW);
  digitalWrite(BLU_PIN, LOW);
  delay(1000);

  // 2. Green Phase
  digitalWrite(RED_PIN, LOW);
  digitalWrite(GRN_PIN, HIGH);
  digitalWrite(BLU_PIN, LOW);
  delay(1000);

  // 3. Blue Phase
  digitalWrite(RED_PIN, LOW);
  digitalWrite(GRN_PIN, LOW);
  digitalWrite(BLU_PIN, HIGH);
  delay(1000);

  // 4. White Phase
  digitalWrite(RED_PIN, HIGH);
  digitalWrite(GRN_PIN, HIGH);
  digitalWrite(BLU_PIN, HIGH);
  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **RGB LED** onto the canvas.
2. Connect RGB LED pins: **Red** to **GP11**, **Common Cathode** (GND) to **GND**, **Green** to **GP12**, **Blue** to **GP13**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Watch the RGB LED cycle between Red, Green, Blue, and White.

## Expected Output

Terminal:
```
Simulation active. Cycling Red -> Green -> Blue -> White.
```

## Expected Canvas Behavior
| Cycle Time | Red Pin (GP11) | Green Pin (GP12) | Blue Pin (GP13) | RGB Color |
| --- | --- | --- | --- | --- |
| 0–1.0 s | HIGH | LOW | LOW | Red |
| 1.0–2.0 s | LOW | HIGH | LOW | Green |
| 2.0–3.0 s | LOW | LOW | HIGH | Blue |
| 3.0–4.0 s | HIGH | HIGH | HIGH | White |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(RED_PIN, HIGH)` | Drives Red channel HIGH while turning Green and Blue channels LOW to start the cycle. |
| `delay(1000)` | Holds each color state for 1 second. |

## Hardware & Safety Concept: Color Mixing Chemistry
Color mixing in LEDs is additive. Unlike subtractive mixing (paints/pigments where mixing everything makes black), additive mixing combines light frequencies. When Red, Green, and Blue light sources merge at equal intensities, they hit the eyes simultaneously, triggering all three cone cell types to register the visual sensation of white.

## Try This! (Challenges)
1. **Secondary Palette**: Add three more steps to the loop to display Yellow (Red+Green), Cyan (Green+Blue), and Magenta (Red+Blue).
2. **Rapid Strobe**: Change the delay times to 100 ms to create a rapid color strobe.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Colors do not mix cleanly | Channels are wired to wrong pins | Check pin order. Swapping GP11 and GP12 will swap the Red and Green cycles. |

## Mode Notes
This multi-output sequencer runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [07 - Pico RGB Red](07-pico-rgb-red.md)
- [08 - Pico RGB Green](08-pico-rgb-green.md)
