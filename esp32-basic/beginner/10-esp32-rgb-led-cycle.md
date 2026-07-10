# 10 - ESP32 RGB LED Cycle

Build a dynamic status indicator that cycles through primary colors (Red → Green → Blue) on an RGB LED.

## Goal
Learn how to program dynamic state changes, manipulate multiple outputs sequentially inside the `loop()` function, and create visual transitions.

## What You Will Build
An RGB LED connected to GPIO 12, 13, and 14 cycles through primary colors (Red, Green, Blue) at a steady 1-second interval.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| RGB LED (Common Cathode) | `rgb_led` | Yes | Yes |
| 220 Ω Resistors × 3 | `resistor` | Optional | Yes (current limit) |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| RGB LED | Red Anode (R) | GPIO12 | Red | Red channel control (via resistor) |
| RGB LED | Green Anode (G) | GPIO13 | Green | Green channel control (via resistor) |
| RGB LED | Blue Anode (B) | GPIO14 | Blue | Blue channel control (via resistor) |
| RGB LED | Common Cathode | GND | Black | Shared ground connection |

> **Wiring tip:** Ground is the longest pin of the RGB LED. Connect it to a GND pin. The three color channels connect to GPIO 12, 13, and 14 respectively.

## Code
```cpp
// Define RGB LED pin mappings
const int RED_PIN = 12;
const int GREEN_PIN = 13;
const int BLUE_PIN = 14;

const int CYCLE_DELAY = 1000; // Time in ms to display each color

void setup() {
  pinMode(RED_PIN, OUTPUT);
  pinMode(GREEN_PIN, OUTPUT);
  pinMode(BLUE_PIN, OUTPUT);
  
  Serial.begin(115200);
  Serial.println("ESP32 RGB Color Cycler Started.");
}

void loop() {
  // 1. Display RED
  digitalWrite(RED_PIN, HIGH);
  digitalWrite(GREEN_PIN, LOW);
  digitalWrite(BLUE_PIN, LOW);
  Serial.println("Color: RED");
  delay(CYCLE_DELAY);
  
  // 2. Display GREEN
  digitalWrite(RED_PIN, LOW);
  digitalWrite(GREEN_PIN, HIGH);
  digitalWrite(BLUE_PIN, LOW);
  Serial.println("Color: GREEN");
  delay(CYCLE_DELAY);
  
  // 3. Display BLUE
  digitalWrite(RED_PIN, LOW);
  digitalWrite(GREEN_PIN, LOW);
  digitalWrite(BLUE_PIN, HIGH);
  Serial.println("Color: BLUE");
  delay(CYCLE_DELAY);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **RGB LED** onto the canvas.
2. Connect RGB LED **R** to **GPIO12**.
3. Connect RGB LED **G** to **GPIO13**.
4. Connect RGB LED **B** to **GPIO14**.
5. Connect RGB LED **GND** to ESP32 **GND**.
6. Paste the C++ code into the editor.
7. Select interpreted C++ mode and click **Run**.

## Expected Output
Serial Monitor:
```
ESP32 RGB Color Cycler Started.
Color: RED
Color: GREEN
Color: BLUE
Color: RED
```

## Expected Canvas Behavior
* The RGB LED lights up Red for 1 second, Green for 1 second, Blue for 1 second, then repeats.
* The terminal logs the active color state in sync with the light.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `digitalWrite(RED_PIN, HIGH);` | Turns the Red channel ON while turning the Green and Blue channels OFF. |
| `delay(CYCLE_DELAY);` | Holds the active color for 1 second before proceeding to the next block. |

## Hardware & Safety Concept: Mixing Colors (Additive Synthesis)
By turning on multiple color channels simultaneously, you can synthesize other colors:
- **Red + Green = Yellow**
- **Red + Blue = Magenta**
- **Green + Blue = Cyan**
- **Red + Green + Blue = White**
This is the principle of additive color mixing used in modern displays and smart lights.

## Try This! (Challenges)
1. **7-Color Cycle**: Expand the `loop()` function to cycle through Red, Green, Blue, Yellow, Magenta, Cyan, and White.
2. **Speed Transition**: Decrease the `CYCLE_DELAY` to 200 ms to create a fast stroboscopic color effect.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Colors are blended or incorrect | Pin mismatch | Check that the pins are wired in the correct order: R to GPIO12, G to GPIO13, B to GPIO14. If Red and Green are swapped, the cycle order will be wrong. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [07 - ESP32 RGB LED Static Red](07-esp32-rgb-led-static-red.md)
- [08 - ESP32 RGB LED Static Green](08-esp32-rgb-led-static-green.md)
- [09 - ESP32 RGB LED Static Blue](09-esp32-rgb-led-static-blue.md)
