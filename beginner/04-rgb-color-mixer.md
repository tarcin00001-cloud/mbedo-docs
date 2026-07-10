# 04 - RGB Color Mixer

Cycle through red, green, blue, and mixed colors using a common-cathode RGB LED.

## Goal
Control three individual color channels (Red, Green, and Blue) of a single RGB LED to produce solid colors and secondary mixed colors.

## What You Will Build
A common-cathode RGB LED connected to PWM pins D9, D10, and D11 cycles through six colors - red, green, blue, yellow, cyan, and magenta - pausing for one second at each color.

**Why D9, D10, and D11?** These three adjacent pins on the Uno all support PWM, which allows for analog brightness control of each color channel individually.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| RGB LED | `rgbled` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes (3 required) |

## Wiring
| RGB LED Pin | Arduino Pin | Notes |
| --- | --- | --- |
| R (Red Anode) | D9 | Red channel control |
| G (Green Anode) | D10 | Green channel control |
| B (Blue Anode) | D11 | Blue channel control |
| C (Common Cathode) | GND | Common ground connection |

> On real hardware, always place a **220 ohm resistor** in series on each of the R, G, and B signal lines to prevent burnout.

## Code
```cpp
const int R_PIN = 9;
const int G_PIN = 10;
const int B_PIN = 11;

void setup() {
  pinMode(R_PIN, OUTPUT);
  pinMode(G_PIN, OUTPUT);
  pinMode(B_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("RGB Color Mixer ready");
}

void loop() {
  // Red
  analogWrite(R_PIN, 255); analogWrite(G_PIN, 0);   analogWrite(B_PIN, 0);
  Serial.println("Red"); delay(1000);

  // Green
  analogWrite(R_PIN, 0);   analogWrite(G_PIN, 255); analogWrite(B_PIN, 0);
  Serial.println("Green"); delay(1000);

  // Blue
  analogWrite(R_PIN, 0);   analogWrite(G_PIN, 0);   analogWrite(B_PIN, 255);
  Serial.println("Blue"); delay(1000);

  // Yellow (Red + Green)
  analogWrite(R_PIN, 255); analogWrite(G_PIN, 255); analogWrite(B_PIN, 0);
  Serial.println("Yellow"); delay(1000);

  // Cyan (Green + Blue)
  analogWrite(R_PIN, 0);   analogWrite(G_PIN, 255); analogWrite(B_PIN, 255);
  Serial.println("Cyan"); delay(1000);

  // Magenta (Red + Blue)
  analogWrite(R_PIN, 255); analogWrite(G_PIN, 0);   analogWrite(B_PIN, 255);
  Serial.println("Magenta"); delay(1000);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** onto the canvas.
2. Drag **RGB LED** onto the canvas.
3. Connect RGB LED **R** to Arduino **D9**.
4. Connect RGB LED **G** to Arduino **D10**.
5. Connect RGB LED **B** to Arduino **D11**.
6. Connect RGB LED **C** to Arduino **GND**.
7. Paste the code into the editor.
8. Use the default Arduino interpreted runtime.
9. Click **Run**.

## Expected Output

Serial Monitor:
```
RGB Color Mixer ready
Red
Green
Blue
Yellow
Cyan
Magenta
...
```

### Expected Canvas Behavior

| Color Mode | Red Pin (D9) | Green Pin (D10) | Blue Pin (D11) | Result |
| --- | --- | --- | --- | --- |
| Red | HIGH (255) | LOW (0) | LOW (0) | Red |
| Green | LOW (0) | HIGH (255) | LOW (0) | Green |
| Blue | LOW (0) | LOW (0) | HIGH (255) | Blue |
| Yellow | HIGH (255) | HIGH (255) | LOW (0) | Yellow |
| Cyan | LOW (0) | HIGH (255) | HIGH (255) | Cyan |
| Magenta | HIGH (255) | LOW (0) | HIGH (255) | Magenta |

The RGB LED on the canvas changes color sequentially every 1 second.

## Code Walkthrough

| Line | What It Does |
| --- | --- |
| `pinMode` (setup) | Sets all three control pins (D9, D10, D11) as outputs. |
| `analogWrite(R_PIN, 255)` | Sets the red channel to 100% duty cycle. |
| `analogWrite(G_PIN, 255)` | Sets the green channel to 100% duty cycle, combining with red to create yellow light. |

## Hardware & Safety Concept: Additive Color Mixing
The **RGB LED** is actually three tiny, independent LEDs (Red, Green, and Blue) packaged into a single plastic housing. 
- A **Common Cathode** RGB LED connects all three negative terminals to a single pin (GND), requiring positive voltage on the R, G, and B pins to light up.
- **Additive Color Mixing** blends these three primary light colors. By adjusting the PWM duty cycle of each pin, you can mix thousands of custom colors (e.g. Red + Green = Yellow).

## Try This! (Challenges)
1. **White Light Mixer**: Mix all three colors to create white light. (Hint: write 255 to all three pins).
2. **Orange & Purple**: Try custom mixtures like Orange (R=255, G=128, B=0) or Purple (R=128, G=0, B=128).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Colors are swapped | R, G, or B wires are crossed | Check the wiring table: R to D9, G to D10, B to D11. |
| LED is completely off | Common cathode (C) pin is not grounded | Ensure the center/longest leg (C) is connected to a GND pin on the Arduino. |

## Mode Notes
These patterns (`analogWrite` calls sequentially in a flat structure) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [03 - Breathing LED](03-breathing-led.md)
- [06 - WS2812B Color Cycle](06-ws2812b-color-cycle.md)
