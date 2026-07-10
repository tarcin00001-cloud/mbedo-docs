# 59 - Pico LCD Custom

Create and print custom glyphs on a 16x2 character LCD using the Pico.

## Goal
Learn how to define custom pixel arrays and store them in the character generator RAM (CGRAM) of the HD44780 LCD controller.

## What You Will Build
A custom symbols display:
- **16x2 I2C LCD (GP4 SDA, GP5 SCL)**: Displays a custom heart symbol and a custom smiley face symbol.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| I2C LCD | VCC | 5V | Power supply |
| I2C LCD | SDA | GP4 | I2C Data |
| I2C LCD | SCL | GP5 | I2C Clock |
| I2C LCD | GND | GND | Ground |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

// Custom heart glyph pixel matrix (8 rows of 5 pixels)
// 1 = pixel ON, 0 = pixel OFF
byte heart[8] = {
  0b00000,
  0b01010,
  0b11111,
  0b11111,
  0b01110,
  0b00100,
  0b00000,
  0b00000
};

// Custom smiley face glyph pixel matrix
byte smiley[8] = {
  0b00000,
  0b00000,
  0b01010,
  0b00000,
  0b10001,
  0b01110,
  0b00000,
  0b00000
};

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  lcd.init();
  lcd.backlight();

  // Load custom glyphs into CGRAM memory locations 0 and 1
  lcd.createChar(0, heart);
  lcd.createChar(1, smiley);

  lcd.setCursor(0, 0);
  lcd.print("Love: ");
  lcd.write(0); // Print custom character 0 (heart)

  lcd.setCursor(0, 1);
  lcd.print("Smile: ");
  lcd.write(1); // Print custom character 1 (smiley)
}

void loop() {
  delay(100);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **I2C LCD Display** onto the canvas.
2. Connect LCD: **VCC** to **5V**, **SDA** to **GP4**, **SCL** to **GP5**, **GND** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Observe the custom heart and smiley glyphs print on the LCD display.

## Expected Output

Terminal:
```
Simulation active. Custom characters loaded to CGRAM.
```

## Expected Canvas Behavior
* Row 0: `Love: [Heart Symbol]`
* Row 1: `Smile: [Smiley Symbol]`

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `lcd.createChar(0, heart)` | Transfers the 8-byte pixel pattern array into index 0 of the HD44780 controller's CGRAM memory block. |
| `lcd.write(0)` | Sends raw byte code 0 to display the custom symbol loaded at CGRAM index 0 (normally non-printable ASCII). |

## Hardware & Safety Concept: Character Generator RAM
Standard character displays contain a Character Generator ROM (CGROM) which holds the bitmap pixels for standard alphanumeric characters (A, B, C, etc.). However, they also contain 64 bytes of Character Generator RAM (CGRAM) which lets you define up to **8 custom characters** (indices 0 to 7) on a 5x8 pixel grid. These characters must be reloaded into CGRAM every time the LCD is powered cycle.

## Try This! (Challenges)
1. **Battery Indicator**: Create a custom battery icon (e.g. 0b01110 for top cap followed by solid rows) and display it on the screen.
2. **Animation**: Alternately write character 0 and character 1 on col 12 of row 0 to create a simple animation loop.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Custom character shows up as garbage pixels | Array initialization error | Ensure the byte array contains exactly 8 values, and each value begins with the binary prefix `0b` representing 5 pixels (e.g. `0b00000` to `0b11111`). |

## Mode Notes
This basic I2C character manipulation project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [58 - Pico LCD Print](58-pico-lcd-print.md)
- [60 - Pico OLED Setup](60-pico-oled-setup.md)
