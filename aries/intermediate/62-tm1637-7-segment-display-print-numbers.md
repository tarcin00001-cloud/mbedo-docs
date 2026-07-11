# 62 - TM1637 7-Segment Display Print Numbers

Display numeric values on a TM1637 4-digit 7-segment display using the VEGA ARIES v3 board.

## Goal
Learn how to interface and control a serial-driven 7-segment display using the TM1637 library, configuring pin parameters and managing brightness levels without using loops inside C++ code.

## What You Will Build
A digital numeric panel that displays the static number `1234` across its 4 segments.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| TM1637 4-Digit Display | `tm1637` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| TM1637 Display | CLK | GPIO 14 | Blue | Clock control pin |
| TM1637 Display | DIO | GPIO 15 | Yellow | Data input/output pin |
| TM1637 Display | VCC | 3V3 | Red | Power supply |
| TM1637 Display | GND | GND | Black | Ground return |

> **Wiring tip:** The TM1637 uses two GPIO lines: CLK on GPIO 14 and DIO on GPIO 15. Verify that connections are solid.

## Code
```cpp
#include <TM1637Display.h>

const int CLK_PIN = 14;
const int DIO_PIN = 15;

TM1637Display display(CLK_PIN, DIO_PIN);

void setup() {
  display.setBrightness(0x0f); // Set display brightness to maximum (range: 0x00 to 0x0f)
  display.showNumberDec(1234);  // Print number 1234
}

void loop() {
  delay(100); // Idle delay as display content is static
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** and **TM1637 Display** components onto the canvas.
2. Wire the TM1637: **CLK** to **GPIO 14**, **DIO** to **GPIO 15**, **VCC** to **3V3**, and **GND** to **GND**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
TM1637 Display configured. Value: 1234
```

## Expected Canvas Behavior
* The TM1637 display lights up showing `1234` on its red 7-segment characters.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `#include <TM1637Display.h>` | Imports the TM1637 7-segment display control library. |
| `TM1637Display display(...)` | Instantiates the display object using clock and data pin parameters. |
| `display.setBrightness(0x0f)` | Configures internal LED driver current intensity parameters (0 to 15 scale). |
| `display.showNumberDec(1234)` | Converts the integer to segment drive bytes and pushes them to the display. |

## Hardware & Safety Concept
* **LED Multiplexing**: To drive four 7-segment displays directly, 32 pins are required. To reduce this, the TM1637 uses time-division multiplexing. It cycles lighting up each digit one by one at high frequency. The human eye cannot detect this due to persistence of vision. This reduces pins to just two.
* **Current Sourcing**: The TM1637 driver chip (a custom ASIC) handles all the multiplexing and constant-current sourcing internally. This prevents the microcontroller pins from being overloaded with the cumulative current drawn by the segments.

## Try This! (Challenges)
1. **Dynamic Counter**: Modify the main loop to count up from 0 to 9999, incrementing by 1 and updating the screen every 500 ms (use a global state variable instead of loops).
2. **Brightness Adjust**: Cycle the brightness setting from 0x00 to 0x0f and back to 0x00 at 200 ms intervals.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Display is completely blank | Missing brightness configuration | You must call `display.setBrightness()` in setup. Without this, the display output circuits remain disabled. |
| Digits show strange patterns | Wrong pin mapping | Verify that CLK connects to GPIO 14 and DIO connects to GPIO 15 (not swapped). |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [63 - TM1637 Counter Display](63-tm1637-counter-display.md)
- [68 - Stepper Motor Position Log (LCD display)](68-stepper-motor-position-log.md)
