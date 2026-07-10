# 58 - ESP32 16x2 I2C LCD Print Text

Display static and dynamic text messages on a 16×2 LCD using an I2C backpack module — the foundation for all LCD-based projects.

## Goal
Learn how to initialise an I2C LCD with the `LiquidCrystal_I2C` library, print fixed and formatted text to both rows, and update the display in real time with a runtime counter.

## What You Will Build
A 16×2 LCD with I2C backpack on the ESP32's I2C bus (GPIO 21 SDA, GPIO 22 SCL). Row 1 displays a project title; Row 2 displays a running seconds counter that updates every second.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 16×2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| I2C LCD Module | VCC | 5V (Vin) | Red | LCD backlight needs 5 V |
| I2C LCD Module | GND | GND | Black | Common ground |
| I2C LCD Module | SDA | GPIO21 | Blue | I2C data line |
| I2C LCD Module | SCL | GPIO22 | Yellow | I2C clock line |

> **Wiring tip:** The default I2C address for most LCD backpacks is `0x27`. If the display shows nothing (no characters, just backlight), try address `0x3F`. You can scan for the actual address using an I2C scanner sketch. GPIO 21 (SDA) and GPIO 22 (SCL) are the default I2C pins on the ESP32 DevKitC. The I2C bus only needs two wires for full bidirectional communication.

## Code
```cpp
// 16x2 I2C LCD Text Display
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// Set LCD I2C address (0x27 or 0x3F) and dimensions
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  lcd.init();          // Initialise the LCD
  lcd.backlight();     // Turn backlight on

  // Row 0 (top) — static title
  lcd.setCursor(0, 0);
  lcd.print("  MbedO ESP32   ");

  Serial.begin(115200);
  Serial.println("LCD ready.");
}

void loop() {
  // Row 1 (bottom) — running seconds counter
  unsigned long seconds = millis() / 1000;

  lcd.setCursor(0, 1);
  lcd.print("Uptime: ");
  lcd.print(seconds);
  lcd.print("s      ");   // Trailing spaces clear old digits

  Serial.print("Uptime: "); Serial.print(seconds); Serial.println("s");

  delay(1000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **16×2 I2C LCD** onto the canvas.
2. Connect LCD **SDA** to **GPIO21**, **SCL** to **GPIO22**.
3. Paste the code and click **Run**.
4. Observe Row 1 showing the title, Row 2 incrementing every second.

## Expected Output
Serial Monitor:
```
LCD ready.
Uptime: 1s
Uptime: 2s
Uptime: 3s
```

LCD Display:
```
  MbedO ESP32
Uptime: 7s
```

## Expected Canvas Behavior
* LCD widget shows the title on the top row from startup.
* The bottom row updates the uptime counter every second.
* The trailing spaces prevent digit artifacts when the number length changes (e.g. 9→10).

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `LiquidCrystal_I2C lcd(0x27, 16, 2)` | Creates an LCD object at I2C address 0x27 with 16 columns and 2 rows. |
| `lcd.init()` | Sends the HD44780 initialisation sequence over I2C. |
| `lcd.backlight()` | Enables the LED backlight transistor on the I2C backpack PCF8574 chip. |
| `lcd.setCursor(0, 1)` | Moves the cursor to column 0, row 1 (bottom row). |
| `lcd.print(seconds)` | Prints the integer uptime value at the current cursor position. |
| `lcd.print("s      ")` | The trailing spaces overwrite any previously longer number — a simple screen-clear technique. |

## Hardware & Safety Concept: I2C LCD Backpack (PCF8574)
A standard 16×2 LCD requires 6 GPIO pins in 4-bit mode (RS, E, D4–D7). An **I2C backpack** adds a PCF8574 I/O expander chip to the LCD, multiplexing all 6 data pins over just 2 I2C wires (SDA + SCL). The PCF8574 receives I2C bytes from the ESP32 and toggles the LCD's parallel pins accordingly — so a complex 6-wire interface is reduced to a 2-wire bus shared by up to 127 devices. The `LiquidCrystal_I2C` library handles the complete HD44780 initialisation sequence, cursor positioning, and character writes over I2C transparently. The LCD VCC should be 5 V for reliable backlight brightness; the I2C signals at 3.3 V are sufficient because the PCF8574 operates from 2.5–6 V.

## Try This! (Challenges)
1. **Scrolling text**: Use `lcd.scrollDisplayLeft()` in a loop to scroll a long message across Row 1.
2. **Two-line message**: Display a custom name on Row 1 and a tagline on Row 2 at startup.
3. **Temperature display**: Read an NTC thermistor (Project 35) and display live temperature on Row 2 instead of uptime.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Backlight on but no text visible | Wrong I2C address | Try `LiquidCrystal_I2C lcd(0x3F, 16, 2)` |
| Display shows garbled squares | VCC too low or missing `lcd.init()` | Ensure VCC is 5 V and `lcd.init()` is the first LCD call in setup |
| Text does not update | Missing `lcd.setCursor()` before each print | Always set cursor position before printing new data |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [59 - ESP32 16×2 I2C LCD Custom Characters](59-esp32-lcd-custom-characters.md)
- [73 - ESP32 DHT22 Temperature LCD HUD](73-esp32-dht22-temperature-lcd-hud.md)
- [66 - ESP32 Servo Position Display](66-esp32-servo-position-display.md)
