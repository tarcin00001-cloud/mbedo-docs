# 115 - Keypad Display HUD

Build a digital typewriter or text entry head-up display (HUD) that prints characters typed on a 4x4 matrix keypad directly onto an I2C LCD display using the VEGA ARIES v3 board.

## Goal
Learn how to capture keypad matrix data streams, accumulate character strings dynamically, and display active input buffers on a character LCD without loops.

## What You Will Build
An interactive input screen. When keys are pressed on the 4x4 keypad, they are printed sequentially on the second row of the LCD screen (next to the label `Type: `). Pressing the `*` key clears the screen, and pressing `#` flashes the LCD backlight to confirm text submission.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Keypad | Row 1 (Pin 1) | GPIO 2 | Red | Keypad Row 1 scan pin |
| Keypad | Row 2 (Pin 2) | GPIO 3 | Orange | Keypad Row 2 scan pin |
| Keypad | Row 3 (Pin 3) | GPIO 4 | Yellow | Keypad Row 3 scan pin |
| Keypad | Row 4 (Pin 4) | GPIO 5 | Green | Keypad Row 4 scan pin |
| Keypad | Col 1 (Pin 5) | GPIO 6 | Blue | Keypad Column 1 scan pin |
| Keypad | Col 2 (Pin 6) | GPIO 7 | Purple | Keypad Column 2 scan pin |
| Keypad | Col 3 (Pin 7) | GPIO 8 | White | Keypad Column 3 scan pin |
| Keypad | Col 4 (Pin 8) | GPIO 9 | Gray | Keypad Column 4 scan pin |
| I2C LCD | VCC | 5V | Red | LCD power supply (5V) |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | SDA0 (GP17) | Blue | I2C Data line |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** Connect the I2C LCD to SDA0 (GP17) and SCL0 (GP16). Ensure the keypad row and column scan lines are wired sequentially to ARIES GPIOs 2 to 9.

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Keypad.h>

// Matrix layout
const byte ROWS = 4;
const byte COLS = 4;

char keys[ROWS][COLS] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};

byte rowPins[ROWS] = {2, 3, 4, 5};   // Keypad row pins
byte colPins[COLS] = {6, 7, 8, 9};   // Keypad column pins

Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);
LiquidCrystal_I2C lcd(0x27, 16, 2);

String typedMessage = "";

void setup() {
  Wire.begin();
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Keypad HUD");
  lcd.setCursor(0, 1);
  lcd.print("Type: ");
}

void loop() {
  char key = keypad.getKey();

  if (key) {
    // '*' acts as clear screen
    if (key == '*') {
      typedMessage = ""; // Reset buffer
      lcd.setCursor(0, 1);
      lcd.print("Type:           "); // Clear existing printed digits
    } 
    // '#' flashes the backlight to indicate message submit
    else if (key == '#') {
      lcd.noBacklight();
      delay(100);
      lcd.backlight();
    } 
    // Append other keys
    else {
      // 16-character limit: "Type: " takes 6 spaces, leaving 10 spaces for inputs
      if (typedMessage.length() < 10) {
        typedMessage += key;
        lcd.setCursor(6, 1);
        lcd.print(typedMessage);
      }
    }
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **4x4 Matrix Keypad**, and **I2C LCD Display** components onto the canvas.
2. Wire the Keypad: Pins **1-4** to **GPIO 2-5**, and Pins **5-8** to **GPIO 6-9**.
3. Wire the LCD: **VCC** to **5V**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
LCD HUD Online.
Input Character: 5
Input Character: A
Input Character: Clear Event
```

## Expected Canvas Behavior
* Pressing standard key buttons (like `1`, `B`, `0`, etc.) causes the corresponding character to display on the LCD screen.
* Pressing `*` instantly clears all typed characters on the LCD.
* Pressing `#` flashes the LCD backlight once.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `makeKeymap(keys)` | Registers the 2D keymap configuration for key decoding. |
| `keypad.getKey()` | Scans the 4x4 matrix, returning the character value of any closed contact. |
| `typedMessage = ""` | Clears the String buffer holding the message characters. |
| `lcd.setCursor(6, 1)` | Moves the LCD cursor to the 7th column of the 2nd row to preserve the `Type: ` prefix. |

## Hardware & Safety Concept
* **Matrix Circuit Safety**: The internal design of a keypad connects row and column traces directly. When multiple keys in the same column are pressed simultaneously, it creates a short circuit path between the row output pins. The `Keypad` library avoids electrical damage by setting inactive scan lines to high-impedance input mode during scanning.
* **Debouncing in Software**: Pressing a button triggers mechanical bounces that generate false duplicate presses. The `Keypad` library uses built-in state debouncing filters, scanning at a set interval to ensure that each physical keypress returns only one character.

## Try This! (Challenges)
1. **Password masking**: Modify the display routine so that instead of displaying the actual typed character (e.g. `1`), the LCD prints an asterisk (`*`) to hide secret codes.
2. **Alert sounder**: Connect a buzzer to GPIO 14. Sound a short 50 ms beep for every key pressed to provide auditory feedback to the user.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD display does not show typing | LCD contrast setting is too low | Adjust the physical potentiometer on the back of the I2C LCD adapter board to adjust contrast. |
| Row of characters overflows and wraps around | Input buffer exceeds screen length | Restrict the string length check `typedMessage.length() < 10` to fit within the 16 character display limits. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [58 - 16x2 I2C LCD Hello World](58-16x2-i2c-lcd-hello-world.md)
- [114 - Keypad 4x4 Password Lock](114-keypad-4x4-password-lock.md)
