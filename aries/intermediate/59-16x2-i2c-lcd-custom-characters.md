# 59 - 16x2 I2C LCD Custom Characters

Define, create, and display custom 5x8 pixel symbols on a 16x2 I2C Character LCD using the VEGA ARIES v3 board.

## Goal
Learn how to define custom bitmaps, write them into the LCD's volatile Character Generator RAM (CGRAM), and display custom shapes using pointers without declaring standard C++ array variables.

## What You Will Build
An LCD screen displaying custom heart and smiley icons side-by-side with text labels on separate rows.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| I2C LCD | VCC | 5V | Red | LCD power supply |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | SDA0 (GP17) | Blue | I2C Data line |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** SDA and SCL connect to ARIES hardware I2C0 pins GP17 and GP16. Ensure the display has 5V power to maintain brightness.

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

// Define structure for custom character patterns 
// to avoid standard array variable declarations ([8]).
struct CustomGlyph {
  byte b0, b1, b2, b3, b4, b5, b6, b7;
};

CustomGlyph heart = {
  0b00000,
  0b01010,
  0b11111,
  0b11111,
  0b01110,
  0b00100,
  0b00000,
  0b00000
};

CustomGlyph smiley = {
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
  Wire.begin();

  lcd.init();
  lcd.backlight();

  // Load custom glyph structures into CGRAM slots 0 and 1
  lcd.createChar(0, (byte*)&heart);
  lcd.createChar(1, (byte*)&smiley);

  lcd.setCursor(0, 0);
  lcd.print("Love: ");
  lcd.write(0); // Display custom character 0 (heart)

  lcd.setCursor(0, 1);
  lcd.print("Smile: ");
  lcd.write(1); // Display custom character 1 (smiley)
}

void loop() {
  delay(100);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** and **I2C LCD Display** components onto the canvas.
2. Wire the LCD: **VCC** to **5V**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Custom glyphs loaded to CGRAM slots 0 and 1.
```

## Expected Canvas Behavior
* The LCD displays `Love: [Heart Icon]` on the first row and `Smile: [Smiley Icon]` on the second row.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `struct CustomGlyph { ... }` | Creates a custom C++ structure with 8 byte variables to hold pixel rows. |
| `lcd.createChar(0, (byte*)&heart)` | Stores the heart bitmap address pointer at CGRAM memory slot 0. |
| `lcd.write(0)` | Tells the display to draw the custom icon stored in CGRAM index 0 at the cursor. |

## Hardware & Safety Concept
* **HD44780 Character Generator RAM**: Standard alphanumeric screens contain character shapes in Character Generator ROM (CGROM). However, they provide 64 bytes of volatile Character Generator RAM (CGRAM) for up to 8 custom symbols. Because this memory is volatile, custom character matrices must be re-registered in CGRAM during each `setup()` execution.
* **Pixel Row Mapping**: Each custom symbol is represented as a grid of 5 columns and 8 rows. In binary format, the 5 least significant bits of each byte correspond to the 5 pixels in a row (e.g. `0b01010` turns on columns 2 and 4 of that row).

## Try This! (Challenges)
1. **Progress Bar Block**: Create a custom block character (e.g. solid block `0b11111` for all rows) to print a progress meter on the LCD.
2. **Heart Animation**: Create a new custom character representing an empty heart shape. Modify the main loop to toggle the character at column 12 between the solid heart and the empty heart every 500 ms.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Custom characters display as garbage blocks | Address casting or missing definition | Ensure you pass the address pointer `(byte*)&heart` correctly in the `createChar()` function call. |
| Display remains blank | I2C address conflict | Verify that the address parameter in the constructor matches your hardware (typically `0x27`). |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [58 - 16x2 I2C LCD Hello World](58-16x2-i2c-lcd-hello-world.md)
- [66 - Servo Position Display](66-servo-position-display.md)
