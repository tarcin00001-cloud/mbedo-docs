# 58 - Pico LCD Print

Print custom text messages to a 16x2 character display using I2C communication.

## Goal
Learn how to initialize I2C buses, address liquid crystal displays, and format character strings on screens.

## What You Will Build
A standard digital greeting screen:
- **16x2 I2C LCD (GP4 SDA, GP5 SCL)**: Displays "Hello, World!" on line 1 and "Raspberry PiPico" on line 2.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes (with PCF8574 chip) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| I2C LCD | VCC | 5V (or VBUS) | 5V required for character contrast |
| I2C LCD | SDA | GP4 (I2C0 SDA) | Data line |
| I2C LCD | SCL | GP5 (I2C0 SCL) | Clock line |
| I2C LCD | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// Set the LCD address to 0x27 or 0x3F for a 16 chars and 2 line display
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  // Initialize standard I2C pins for Pico: SDA = GP4, SCL = GP5
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  lcd.init();
  lcd.backlight();
  
  lcd.setCursor(0, 0); // Column 0, Row 0
  lcd.print("Hello, World!");
  
  lcd.setCursor(0, 1); // Column 0, Row 1
  lcd.print("Raspberry PiPico");
}

void loop() {
  // Static display - nothing to update in loop
  delay(100);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **I2C LCD Display** onto the canvas.
2. Connect LCD: **VCC** to **5V**, **SDA** to **GP4**, **SCL** to **GP5**, **GND** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Observe the characters "Hello, World!" and "Raspberry PiPico" print on the screen.

## Expected Output

Terminal:
```
Simulation active. LCD initialized at address 0x27.
```

## Expected Canvas Behavior
* Row 0: `Hello, World!`
* Row 1: `Raspberry PiPico`

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Wire.setSDA(4); Wire.setSCL(5);` | Tells the Pico's I2C peripheral hardware block to route SDA and SCL to GP4 and GP5. |
| `LiquidCrystal_I2C lcd(0x27, 16, 2)` | Configures the display object to target address 0x27 with 16 columns and 2 rows. |
| `lcd.setCursor(0, 1)` | Moves the drawing cursor to the first character of the second row. |

## Hardware & Safety Concept: I2C Pull-Up Resistors
I2C is a open-drain bus, meaning the master and slaves can only pull the lines LOW (to GND). They cannot drive the lines HIGH. Pulling the lines HIGH relies on external pull-up resistors (typically 4.7k ohms) tied to VCC. Without these pull-up resistors, the SDA and SCL lines float and remain LOW, preventing any communication. Most hobby I2C LCD backpack modules contain these pull-ups pre-installed.

## Try This! (Challenges)
1. **Flashing Backlight**: Modify the loop to turn the LCD backlight ON and OFF every 1 second.
2. **Text Shift**: Change row 2 to print your name, centered on the screen.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD backlight is ON but no text is visible | Contrast dial is misaligned | Use a small flathead screwdriver to turn the blue potentiometer dial on the back of the physical I2C backpack board to adjust contrast. |

## Mode Notes
This basic I2C communication project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [59 - Pico LCD Custom](59-pico-lcd-custom.md)
- [60 - Pico OLED Setup](60-pico-oled-setup.md)
