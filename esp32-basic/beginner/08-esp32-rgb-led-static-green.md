# 08 - ESP32 RGB LED Static Green

Build a static green status indicator using a common-cathode RGB LED.

## Goal
Learn how to interface an RGB LED with the ESP32, identify green anode configurations, and drive digital outputs to display green.

## What You Will Build
An RGB LED connected to GPIO 12 (Red), 13 (Green), and 14 (Blue) illuminates static Green by powering the green anode pin (GPIO 13) HIGH and holding the red and blue pins LOW.

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

> **Wiring tip:** Connect the Green Anode pin (G) of the RGB LED to GPIO13. The longest pin must connect directly to GND.

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
  Serial.println("ESP32 Static Green Indicator Ready.");
  
  // Set Static Green state: Green HIGH, Red/Blue LOW
  digitalWrite(RED_PIN, LOW);
  digitalWrite(GREEN_PIN, HIGH);
  digitalWrite(BLUE_PIN, LOW);
  
  Serial.println("RGB Output: GREEN");
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
ESP32 Static Green Indicator Ready.
RGB Output: GREEN
```

## Expected Canvas Behavior
* The RGB LED lights up bright green.
* The state is logged once in the Serial Monitor.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `const int GREEN_PIN = 13;` | Maps the Green channel of the RGB LED to GPIO 13. |
| `digitalWrite(GREEN_PIN, HIGH);` | Supplies 3.3V to the Green anode, lighting the internal green diode. |

## Hardware & Safety Concept: Voltage Drops Across Colors
Different LED colors require different voltages to turn on (called **forward voltage drop**, $V_f$):
- **Red LED**: $V_f \approx 1.8\text{ V}$ to $2.0\text{ V}$
- **Green/Blue LED**: $V_f \approx 3.0\text{ V}$ to $3.3\text{ V}$
Because the green and blue diodes require more voltage, they will draw less current than a red diode when connected to the same 3.3V source via identical 220 Ω resistors. This is why the red channel often appears brighter than the green or blue channel unless larger resistors are used on the red line to balance them.

## Try This! (Challenges)
1. **Static Cyan**: Modify the code to display Cyan (Green + Blue HIGH, Red LOW).
2. **Static White**: Modify the code to display White (all channels HIGH).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED glows faint red instead of green | Swapped anodes | Check your wiring. Verify that GPIO 13 is connected to the Green pin (G) of the RGB LED, not the Red pin (R). |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [07 - ESP32 RGB LED Static Red](07-esp32-rgb-led-static-red.md)
- [09 - ESP32 RGB LED Static Blue](09-esp32-rgb-led-static-blue.md)
