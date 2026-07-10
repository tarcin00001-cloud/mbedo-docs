# 09 - ESP32 RGB LED Static Blue

Build a static blue status indicator using a common-cathode RGB LED.

## Goal
Learn how to interface an RGB LED with the ESP32, identify blue anode configurations, and drive digital outputs to display blue.

## What You Will Build
An RGB LED connected to GPIO 12 (Red), 13 (Green), and 14 (Blue) illuminates static Blue by powering the blue anode pin (GPIO 14) HIGH and holding the red and green pins LOW.

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

> **Wiring tip:** Connect the Blue Anode pin (B) of the RGB LED to GPIO14. The longest pin must connect directly to GND.

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
  Serial.println("ESP32 Static Blue Indicator Ready.");
  
  // Set Static Blue state: Blue HIGH, Red/Green LOW
  digitalWrite(RED_PIN, LOW);
  digitalWrite(GREEN_PIN, LOW);
  digitalWrite(BLUE_PIN, HIGH);
  
  Serial.println("RGB Output: BLUE");
}

void loop() {
  // Static state - nothing to execute in loop
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
ESP32 Static Blue Indicator Ready.
RGB Output: BLUE
```

## Expected Canvas Behavior
* The RGB LED lights up bright blue.
* The state is logged once in the Serial Monitor.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `const int BLUE_PIN = 14;` | Maps the Blue channel of the RGB LED to GPIO 14. |
| `digitalWrite(BLUE_PIN, HIGH);` | Supplies 3.3V to the Blue anode, lighting the internal blue diode. |

## Hardware & Safety Concept: Current Limits and Blue LEDs
Blue LEDs are made of Indium Gallium Nitride (InGaN), which has a relatively high forward voltage drop ($V_f \approx 3.2\text{ V}$). When driven by a 3.3V GPIO pin on the ESP32, the voltage difference across the current-limiting resistor is very small:
\[V_{resistor} = 3.3\text{ V} - 3.2\text{ V} = 0.1\text{ V}\]
Even a small 220 Ω resistor will restrict current to under $0.5\text{ mA}$. This is safe, but on physical hardware, the blue channel might appear dimmer than the red channel. Adjusting the resistor value to 100 Ω on the blue channel (or utilizing PWM brightness scaling) helps balance the output colors.

## Try This! (Challenges)
1. **Static Purple**: Modify the code to display Purple (Red + Blue HIGH, Green LOW).
2. **Static Cyan**: Modify the code to display Cyan (Green + Blue HIGH, Red LOW).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED glows green instead of blue | Swapped pin connections | Check your wiring. Verify that GPIO 14 is connected to the Blue pin (B) of the RGB LED, not the Green pin (G). |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [07 - ESP32 RGB LED Static Red](07-esp32-rgb-led-static-red.md)
- [08 - ESP32 RGB LED Static Green](08-esp32-rgb-led-static-green.md)
- [10 - ESP32 RGB LED Cycle](10-esp32-rgb-led-cycle.md)
