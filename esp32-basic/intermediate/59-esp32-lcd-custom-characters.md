# 59 - ESP32 16×2 I2C LCD Custom Characters

Define and display custom 5×8 pixel characters on a 16×2 I2C LCD using the HD44780's CGRAM — creating icons, arrows, and symbols not in the default character set.

## Goal
Learn how to define custom 5×8 bitmaps, load them into the LCD's Character Generator RAM (CGRAM), and display them alongside standard text to build richer UI elements.

## What You Will Build
A 16×2 I2C LCD displaying a progress bar built from custom block characters, a heart icon, and a temperature thermometer symbol — all defined as 5×8 pixel arrays.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 16×2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| I2C LCD Module | VCC | 5V (Vin) | Red | LCD backlight supply |
| I2C LCD Module | GND | GND | Black | Common ground |
| I2C LCD Module | SDA | GPIO21 | Blue | I2C data |
| I2C LCD Module | SCL | GPIO22 | Yellow | I2C clock |

> **Wiring tip:** Custom characters are stored in the HD44780's CGRAM — a volatile 64-byte RAM that holds up to 8 custom characters (indices 0–7). They are lost when power is removed and must be reloaded in `setup()` every time. Define them before calling `lcd.print()` with their index.

## Code
```cpp
// LCD Custom Characters — heart, arrow, thermometer, progress blocks
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

// Custom character bitmaps (5 columns × 8 rows, 0 = off, 1 = on)
byte heart[8]  = {0b00000,0b01010,0b11111,0b11111,0b01110,0b00100,0b00000,0b00000};
byte thermo[8] = {0b00100,0b01010,0b01010,0b01110,0b01110,0b11111,0b11111,0b01110};
byte arrow[8]  = {0b00000,0b00100,0b00110,0b11111,0b00110,0b00100,0b00000,0b00000};
byte block[8]  = {0b11111,0b11111,0b11111,0b11111,0b11111,0b11111,0b11111,0b11111};

void setup() {
  lcd.init();
  lcd.backlight();

  // Load custom characters into CGRAM slots 0–3
  lcd.createChar(0, heart);
  lcd.createChar(1, thermo);
  lcd.createChar(2, arrow);
  lcd.createChar(3, block);

  // Row 0: title with heart icon
  lcd.setCursor(0, 0);
  lcd.write(byte(0));          // Heart character (slot 0)
  lcd.print(" MbedO ESP32 ");
  lcd.write(byte(0));          // Heart character again

  Serial.begin(115200);
  Serial.println("Custom Character LCD ready.");
}

void loop() {
  // Row 1: animated progress bar (0–16 blocks, cycling)
  static int progress = 0;
  lcd.setCursor(0, 1);

  for (int i = 0; i < 16; i++) {
    if (i < progress) {
      lcd.write(byte(3));   // Filled block
    } else {
      lcd.print(" ");       // Empty space
    }
  }

  Serial.print("Progress: "); Serial.print(progress); Serial.println("/16");

  progress = (progress + 1) % 17;   // 0–16 then reset
  delay(300);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **16×2 I2C LCD** onto the canvas.
2. Connect LCD **SDA** to **GPIO21**, **SCL** to **GPIO22**.
3. Paste the code and click **Run**.
4. Observe the top row showing a heart-flanked title; the bottom row shows a growing progress bar.

## Expected Output
Serial Monitor:
```
Custom Character LCD ready.
Progress: 0/16
Progress: 1/16
Progress: 8/16
Progress: 16/16
Progress: 0/16
```

LCD Display (after 8 steps):
```
♥ MbedO ESP32 ♥
████████
```

## Expected Canvas Behavior
* Row 1 (top) shows heart icons flanking the title text from startup.
* Row 2 (bottom) shows a progress bar growing one block every 300 ms from empty to full, then resetting.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `byte heart[8] = {0b00000,...}` | Defines an 8-row bitmap for the heart character — each byte is one 5-bit row. |
| `lcd.createChar(0, heart)` | Writes the heart bitmap into CGRAM slot 0. |
| `lcd.write(byte(0))` | Prints the custom character stored in CGRAM slot 0 at the cursor position. |
| `lcd.write(byte(3))` | Prints the solid block character (slot 3) as a progress bar segment. |
| `progress = (progress + 1) % 17` | Cycles the progress counter from 0 to 16 then resets to 0. |

## Hardware & Safety Concept: HD44780 Character Generator RAM (CGRAM)
The HD44780 LCD controller stores its font in **Character Generator ROM (CGROM)** — a fixed set of ASCII and international characters. Alongside CGROM, it provides 64 bytes of **CGRAM** for user-defined characters. Each custom character occupies 8 bytes (one per pixel row), giving 5×8 pixels of addressable dots. With 64 bytes total, up to **8 custom characters** (addresses 0–7) can be loaded at once. This feature enables LCDs to display battery level icons, signal bars, custom logos, musical notes, degree symbols, and language-specific glyphs not present in the standard font. The characters remain in RAM until power is removed — they must be reloaded in each `setup()` call.

## Try This! (Challenges)
1. **Battery icon**: Design a 5-level battery indicator (full, ¾, ½, ¼, empty) using 5 CGRAM slots and display it based on a potentiometer reading.
2. **Temperature thermometer**: Display the thermometer custom character (slot 1) followed by a live temperature from an NTC sensor.
3. **Animated icon**: Cycle between two custom character frames (e.g., open eye / closed eye) by alternately loading different bitmaps into the same CGRAM slot.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Custom characters show as squares | `createChar()` called after first `print()` | Always call `createChar()` before any `lcd.print()` or `lcd.write()` |
| Wrong character displayed | Wrong CGRAM slot index | Ensure `lcd.createChar(n, bitmap)` and `lcd.write(byte(n))` use the same index `n` |
| Only 8 custom characters work | CGRAM limit | The HD44780 supports a maximum of 8 custom characters (slots 0–7) simultaneously |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [58 - ESP32 16×2 I2C LCD Print Text](58-esp32-16x2-i2c-lcd-print-text.md)
- [60 - ESP32 OLED SSD1306 Display Setup](60-esp32-oled-ssd1306-display-setup.md)
- [73 - ESP32 DHT22 Temperature LCD HUD](73-esp32-dht22-temperature-lcd-hud.md)
