# 58 - 16x2 I2C LCD Hello World

Print a static greeting message on a 16x2 I2C Character Liquid Crystal Display using the VEGA ARIES v3 board.

## Goal
Learn how to initialize I2C communication, address I2C slave devices, and display characters on a 16x2 LCD using the LiquidCrystal_I2C library in MbedO interpreted mode.

## What You Will Build
A digital greeting panel displaying "Hello, World!" on the first row and "ARIES RISC-V" on the second row of a 16x2 character display.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes (with PCF8574 module) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| I2C LCD | VCC | 5V | Red | LCD power supply |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | SDA0 (GP17) | Blue | I2C Data line |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** The SDA and SCL pins on the LCD connect to the hardware I2C0 interface on the ARIES board: SDA0 (GP17) and SCL0 (GP16). Verify that connections are solid.

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Wire.begin(); // Initialize default I2C0 interface on GP17 (SDA) and GP16 (SCL)

  lcd.init();
  lcd.backlight();

  lcd.setCursor(0, 0); // Column 0, Row 0
  lcd.print("Hello, World!");

  lcd.setCursor(0, 1); // Column 0, Row 1
  lcd.print("ARIES RISC-V");
}

void loop() {
  delay(100); // Idle delay as the display content is static
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** and **I2C LCD Display** components onto the canvas.
2. Connect the LCD: **VCC** to **5V**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
I2C LCD Initialized at Address 0x27.
```

## Expected Canvas Behavior
* The 16x2 LCD screen displays `Hello, World!` on the top row and `ARIES RISC-V` on the bottom row.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Wire.begin()` | Starts the I2C peripheral master clock and data lines. |
| `LiquidCrystal_I2C lcd(0x27, 16, 2)` | Creates an LCD driver object targeting I2C address 0x27, 16 columns wide, 2 rows high. |
| `lcd.init()` | Triggers the display controller setup sequence (sets 4-bit mode, display control, configuration). |
| `lcd.backlight()` | Sends the enable byte to power the display back-lighting LED. |
| `lcd.setCursor(0, 0)` | Positions the character output cursor at column index 0 of row index 0. |

## Hardware & Safety Concept
* **I2C Protocol**: Inter-Integrated Circuit (I2C) is a synchronous, multi-master/multi-slave, packet-switched, single-ended, serial communication bus. It uses two lines: Serial Data (SDA) and Serial Clock (SCL). Each slave device has a unique address (like `0x27` or `0x3F` for character LCDs).
* **Level Shifting**: Character LCD backlights run on 5V, but their PCF8574 expanders can accept 3.3V logic signals from the ARIES board. If using 5V I2C sensors, level shifters are required to prevent overvoltage on the SoC I2C pins.

## Try This! (Challenges)
1. **Flashing Backlight**: Modify the main loop to toggle the LCD backlight ON and OFF every 1 second (use `lcd.noBacklight()` and `lcd.backlight()`).
2. **Text Scrolling**: Use the `lcd.scrollDisplayLeft()` function in the main loop to scroll the text across the display.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Screen is powered but shows black rectangles | Contrast is misaligned | Use a screwdriver to adjust the blue potentiometer on the back of the LCD backpack. |
| Compile error on I2C address | Address mismatch | Check the I2C expander chip; standard PCF8574T chips default to `0x27`, while PCF8574AT chips default to `0x3F`. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [59 - 16x2 I2C LCD Custom Characters](59-16x2-i2c-lcd-custom-characters.md)
- [66 - Servo Position Display](66-servo-position-display.md)
