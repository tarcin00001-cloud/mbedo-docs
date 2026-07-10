# 115 - ESP32 Keypad Display HUD

Build a keypad display console that prints typed characters directly onto a 16x2 I2C LCD screen, supporting input buffers, character clearing, and submit confirmations.

## Goal
Learn how to capture keystrokes from a matrix keypad, update text on specific LCD coordinates, and handle screen clears.

## What You Will Build
A 4x4 matrix keypad is connected to GPIO pins 12, 13, 14, 15, 16, 17, 25, and 26. A 16x2 I2C LCD displays the inputs. Characters typed are printed on Row 2 with masking. Pressing '*' clears the display; pressing '#' submits the string, showing it on the top row.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Keypad Row 1 | Pin 1 | GPIO12 | Red | Keypad rows |
| Keypad Row 2 | Pin 2 | GPIO13 | Orange | Keypad rows |
| Keypad Row 3 | Pin 3 | GPIO14 | Yellow | Keypad rows |
| Keypad Row 4 | Pin 4 | GPIO15 | Green | Keypad rows |
| Keypad Col 1 | Pin 5 | GPIO16 | Blue | Keypad columns |
| Keypad Col 2 | Pin 6 | GPIO17 | Purple | Keypad columns |
| Keypad Col 3 | Pin 7 | GPIO25 | White | Keypad columns |
| Keypad Col 4 | Pin 8 | GPIO26 | Gray | Keypad columns |
| I2C LCD | VCC / GND | 5V / GND | Red / Black | LCD Power |
| I2C LCD | SDA | GPIO21 | Blue | I2C Data |
| I2C LCD | SCL | GPIO22 | Yellow | I2C Clock |

> **Wiring tip:** Share the common I2C bus pins for the LCD. Connect the 8 keypad pins to the corresponding GPIOs.

## Code
```cpp
// Keypad Display HUD
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Keypad.h>

// Keypad configuration
const byte ROWS = 4;
const byte COLS = 4;

char keys[ROWS][COLS] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};

byte rowPins[ROWS] = {12, 13, 14, 15};
byte colPins[COLS] = {16, 17, 25, 26};

Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);
LiquidCrystal_I2C lcd(0x27, 16, 2);

String inputBuffer = "";

void setup() {
  Serial.begin(115200);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Keypad HUD Ready");
  lcd.setCursor(0, 1);
  lcd.print("Type message...");
  
  delay(1500);
  lcd.clear();
}

void loop() {
  char key = keypad.getKey();
  
  if (key) {
    Serial.print("Key: "); Serial.println(key);
    
    // '#' acts as the SUBMIT key
    if (key == '#') {
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Submitted Value:");
      lcd.setCursor(0, 1);
      lcd.print(inputBuffer);
      
      Serial.print("Submitted: "); Serial.println(inputBuffer);
      inputBuffer = ""; // Reset buffer
    } 
    // '*' acts as the CLEAR key
    else if (key == '*') {
      inputBuffer = "";
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Buffer Cleared");
      delay(800);
      lcd.clear();
    } 
    // Append standard characters to input string (limit 16 chars for LCD row width)
    else {
      if (inputBuffer.length() < 16) {
        inputBuffer += key;
        
        // Show current typed characters
        lcd.setCursor(0, 0);
        lcd.print("Inputting...    ");
        lcd.setCursor(0, 1);
        lcd.print(inputBuffer);
        lcd.print("                "); // Clear old digits
      }
    }
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **4x4 Keypad**, and **16x2 I2C LCD** onto the canvas.
2. Connect the 8 keypad pins to **12, 13, 14, 15, 16, 17, 25, 26** and the LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Click characters on the keypad widget and watch them print on the LCD display. Click `#` to submit.

## Expected Output
Serial Monitor:
```
Key: 1
Key: 5
Key: A
Key: #
Submitted: 15A
```

LCD Display (during typing):
```
Inputting...
15A
```

## Expected Canvas Behavior
* Characters clicked on the virtual keypad widget appear immediately on the bottom row of the LCD.
* Pressing the `#` key clears the screen and displays "Submitted Value: [text]".
* Pressing `*` displays "Buffer Cleared" and zeroes the entry line.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `inputBuffer += key` | Appends the pressed key character to the string buffer. |
| `inputBuffer.length() < 16` | Keeps the buffer length within the 16 character boundary of the LCD line width. |
| `lcd.print("                ")` | Appends blank spaces to overwrite any old characters. |

## Hardware & Safety Concept: Display Masking and Buffer Overflows
Secure keypad interfaces (like ATMs) mask character entries with asterisks (`*`) to hide sensitive passcodes from bystanders. Furthermore, input buffers must have strict length constraints (e.g. `< 16`) to prevent memory buffer overflows where entering very long strings causes memory corruption.

## Try This! (Challenges)
1. **Passcode Masking**: Modify the code to display `*` characters on Row 2 while typing, hiding the actual characters until submitted.
2. **Key Sound Tone**: Add a passive buzzer on GPIO 4 that plays a short beep (50ms) for each key pressed.
3. **Key Calculator HUD**: Write a basic calculator that reads two numbers and performs arithmetic when operators (A=+, B=-, C=*, D=/) are pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Keys display incorrect values | Keymap grid coordinates reversed | Check the `rowPins` and `colPins` assignments against the keypad model |
| LCD does not update | I2C address conflict | Verify address `0x27` is matching |
| Characters drop when typed fast | Loop delay is too large | Keep loop execution fast; remove any unnecessary `delay()` blocks |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [114 - ESP32 Keypad 4x4 Password Lock](114-esp32-keypad-4x4-password-lock.md)
- [58 - ESP32 16×2 I2C LCD Print Text](58-esp32-16x2-i2c-lcd-print-text.md)
- [111 - ESP32 Rotary Encoder Menu Selector](111-esp32-rotary-encoder-menu-selector.md)
