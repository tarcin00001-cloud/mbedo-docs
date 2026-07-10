# 62 - Pico TM1637 Print

Display numeric readings on a TM1637 4-digit 7-segment display.

## Goal
Learn how to interface specialized segment drivers, transmit serial data, and display numeric characters on LED screens.

## What You Will Build
A digital numeric panel display:
- **TM1637 Display (CLK GP16, DIO GP17)**: Displays the static number `1234` on its 4 segments.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| TM1637 4-Digit Display | `tm1637` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| TM1637 | VCC | 3.3V (or 5V) | Power supply |
| TM1637 | CLK | GP16 | Clock control |
| TM1637 | DIO | GP17 | Data input/output |
| TM1637 | GND | GND | Ground return |

## Code
```cpp
#include <TM1637Display.h>

const int CLK_PIN = 16;
const int DIO_PIN = 17;

TM1637Display display(CLK_PIN, DIO_PIN);

void setup() {
  display.setBrightness(0x0f); // Set brightness to maximum
  display.showNumberDec(1234);  // Print number 1234
}

void loop() {
  // Static state - nothing to update in loop
  delay(100);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **TM1637 Display** onto the canvas.
2. Connect TM1637: **VCC** to **3V3**, **CLK** to **GP16**, **DIO** to **GP17**, **GND** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Observe the digits `1234` light up on the 7-segment display.

## Expected Output

Terminal:
```
Simulation active. TM1637 displaying 1234.
```

## Expected Canvas Behavior
* TM1637 panel digits show: `1234`

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `display.setBrightness(0x0f)` | Configures the display intensity parameter (0x00 to 0x0f) via the TM1637 controller chip registers. |
| `display.showNumberDec(1234)` | Converts the integer 1234 into segment-activation byte codes and transmits them to the display. |

## Hardware & Safety Concept: LED Multiplexing
Driving four 7-segment displays directly would require 32 GPIO pins (8 segments * 4 digits). To save pins, the TM1637 uses **time-division multiplexing**. It rapidly switches between lighting up each digit one by one at high frequency. The human eye cannot detect this fast switching (due to persistence of vision) and perceives all four digits as glowing continuously. This reduces the pin requirement to just two lines (CLK and DIO).

## Try This! (Challenges)
1. **Dynamic Counter**: Modify the loop to increment a number from 0 to 9999, updating the display every 100 ms.
2. **Colons toggle**: Enable display colons to blink like a clock second ticker.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Screen is blank | Library not initialized | Verify you call `display.setBrightness()` in setup. Without setting brightness, the TM1637 outputs stay disabled. |

## Mode Notes
This basic display project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [58 - Pico LCD Print](58-pico-lcd-print.md)
- [63 - Pico TM1637 Counter](63-pico-tm1637-counter.md)
