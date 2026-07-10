# 07 - ESP32 RGB LED Static Red

Build a static red status indicator using a common-cathode RGB LED.

## Goal
Learn how to interface an RGB (Red-Green-Blue) LED with the ESP32, identify pin configurations, and set static color states using basic digital output writes.

## What You Will Build
An RGB LED connected to GPIO 12 (Red), 13 (Green), and 14 (Blue) illuminates static Red by powering the red anode pin and holding the green and blue pins LOW.

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

> **Wiring tip:** A common-cathode RGB LED features four pins. The longest pin is the shared ground (Cathode). Connect it to ESP32 GND. The other three pins are separate anodes for Red, Green, and Blue.

## Code
```cpp
// Define RGB LED pin mappings
const int RED_PIN = 12;
const int GREEN_PIN = 13;
const int BLUE_PIN = 14;

void setup() {
  // Configure all color channels as outputs
  pinMode(RED_PIN, OUTPUT);
  pinMode(GREEN_PIN, OUTPUT);
  pinMode(BLUE_PIN, OUTPUT);
  
  Serial.begin(115200);
  Serial.println("ESP32 Static Red Indicator Ready.");
  
  // Set Static Red state: Red HIGH, Green/Blue LOW
  digitalWrite(RED_PIN, HIGH);
  digitalWrite(GREEN_PIN, LOW);
  digitalWrite(BLUE_PIN, LOW);
  
  Serial.println("RGB Output: RED");
}

void loop() {
  // Nothing to change in loop - static display
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **RGB LED** onto the canvas.
2. Connect RGB LED **R** to **GPIO12**.
3. Connect RGB LED **G** to **GPIO13**.
4. Connect RGB LED **B** to **GPIO14**.
5. Connect RGB LED **GND** to ESP32 **GND**.
6. Paste code, select interpreted C++ mode, and click **Run**.

## Expected Output
Serial Monitor:
```
ESP32 Static Red Indicator Ready.
RGB Output: RED
```

## Expected Canvas Behavior
* The RGB LED lights up bright red.
* The state is logged once in the Serial Monitor.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `const int RED_PIN = 12;` | Maps the Red channel of the RGB LED to GPIO 12. |
| `digitalWrite(RED_PIN, HIGH);` | Supplies 3.3V to the Red anode, activating the red internal diode. |
| `digitalWrite(GREEN_PIN, LOW);` | Keeps the Green channel off by holding it at 0V. |

## Hardware & Safety Concept: Common-Cathode vs Common-Anode LEDs
- **Common-Cathode (Active-HIGH)**: The negative cathode of all three internal LEDs (R, G, B) is connected to a single pin, which must go to GND. Toggling a GPIO pin HIGH (3.3V) turns that color channel ON.
- **Common-Anode (Active-LOW)**: The positive anode of all three internal LEDs is connected to a single pin, which must go to 3.3V/5V. Toggling a GPIO pin LOW (0V) turns that color channel ON.
*Always confirm which module type you have before writing code.*

## Try This! (Challenges)
1. **Static Yellow**: Modify the code to display Yellow (Red + Green HIGH, Blue LOW).
2. **Static Magenta**: Modify the code to display Magenta (Red + Blue HIGH, Green LOW).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays off or shows wrong color | Common pin wired incorrectly | Ensure the longest pin (common cathode) is connected to GND. If your LED is a common-anode type, connect the long pin to 3.3V and write LOW to the color pins to turn them ON. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [08 - ESP32 RGB LED Static Green](08-esp32-rgb-led-static-green.md)
- [10 - ESP32 RGB LED Cycle](10-esp32-rgb-led-cycle.md)
