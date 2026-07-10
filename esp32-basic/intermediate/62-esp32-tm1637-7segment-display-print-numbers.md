# 62 - ESP32 TM1637 7-Segment Display Print Numbers

Drive a 4-digit 7-segment LED display module (TM1637) to show numerical values — counters, sensor readings, and scores — using a simple 2-wire protocol.

## Goal
Learn how to wire a TM1637 display module, use the TM1637Display library to print integers and set brightness, and build a live seconds counter displayed as a 4-digit number.

## What You Will Build
A TM1637 4-digit display on GPIO 18 (CLK) and GPIO 19 (DIO). The display shows a running seconds counter from 0000 to 9999 with a colon separator, refreshing every second.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| TM1637 4-Digit 7-Segment Display | `tm1637` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| TM1637 Module | VCC | 3V3 | Red | Module power (3.3 V or 5 V) |
| TM1637 Module | GND | GND | Black | Common ground |
| TM1637 Module | CLK | GPIO18 | Yellow | Clock line |
| TM1637 Module | DIO | GPIO19 | Blue | Bidirectional data line |

> **Wiring tip:** The TM1637 uses a proprietary 2-wire protocol (not standard I2C). CLK and DIO are open-drain lines — the module includes pull-up resistors on the PCB, so no external pull-ups are needed. The module accepts 3.3 V or 5 V on VCC; use 3V3 for ESP32 compatibility without level shifting.

## Code
```cpp
// TM1637 4-Digit 7-Segment Display — seconds counter
#include <TM1637Display.h>

const int CLK_PIN = 18;
const int DIO_PIN = 19;

TM1637Display display(CLK_PIN, DIO_PIN);

void setup() {
  display.setBrightness(5);   // Brightness 0 (dim) – 7 (brightest)
  display.clear();

  Serial.begin(115200);
  Serial.println("TM1637 Display ready.");
}

void loop() {
  int seconds = (millis() / 1000) % 10000;   // 0–9999

  // Display the counter with leading zeros and colon
  display.showNumberDecEx(seconds, 0b01000000, true);
  // 0b01000000 = colon ON; true = leading zeros

  Serial.print("Display: "); Serial.println(seconds);
  delay(1000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **TM1637 Display** onto the canvas.
2. Connect TM1637 **CLK** to **GPIO18**, **DIO** to **GPIO19**.
3. Paste the code and click **Run**.
4. Watch the 4-digit counter increment every second on the display widget.

## Expected Output
Serial Monitor:
```
TM1637 Display ready.
Display: 0
Display: 1
Display: 9
Display: 10
Display: 9999
Display: 0
```

TM1637 Display shows:
```
0000 → 0001 → 0002 … → 9999 → 0000
```

## Expected Canvas Behavior
* Display widget shows `0000` at startup with colon lit between digits 2 and 3.
* Increments by 1 each second with leading zeros maintained.
* Rolls over from 9999 to 0000 automatically.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `display.setBrightness(5)` | Sets display brightness to level 5 of 7 — adjust 0–7 as needed. |
| `(millis() / 1000) % 10000` | Converts milliseconds to seconds and wraps at 9999 using modulo. |
| `showNumberDecEx(seconds, 0b01000000, true)` | Displays the integer with: colon enabled (`0b01000000` sets the colon bit), leading zeros (`true`). |

## Hardware & Safety Concept: TM1637 LED Driver Protocol
The TM1637 uses a synchronous serial protocol similar to I2C but with its own framing. Each digit is encoded as a 7-bit segment pattern (segments a–g) and written to one of four digit registers inside the TM1637 chip. The chip then PWM-drives all 28 LED segments (4 digits × 7 segments) using its internal constant-current drivers, ensuring uniform brightness regardless of the number of segments lit. The **constant current drive** eliminates the segment brightness variation that occurs when directly driving 7-segment displays with GPIO pins (where segments with more current-sharing appear dimmer). TM1637 displays are widely used in kitchen timers, digital clocks, lab instruments, and scoreboards.

## Try This! (Challenges)
1. **Countdown timer**: Display a countdown from 9999 to 0000, then flash the display when it reaches zero.
2. **Potentiometer display**: Map a potentiometer (GPIO 34) reading to 0–9999 and show it live on the display.
3. **Temperature display**: Show a DHT22 temperature reading (e.g. 25.5 °C) as `2550` (multiplied by 100 to preserve decimals).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Display stays off | CLK and DIO swapped | Swap GPIO18 and GPIO19 wires on the module |
| Display shows random segments | Incorrect library function call | Use `showNumberDecEx()` not `showNumberDec()` for colon control |
| Brightness very dim | Brightness set to 0 | Call `display.setBrightness(5)` in setup |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [63 - ESP32 TM1637 Counter Display](63-esp32-tm1637-counter-display.md)
- [58 - ESP32 16×2 I2C LCD Print Text](58-esp32-16x2-i2c-lcd-print-text.md)
- [60 - ESP32 OLED SSD1306 Display Setup](60-esp32-oled-ssd1306-display-setup.md)
