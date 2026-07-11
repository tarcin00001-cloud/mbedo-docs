# 111 - Rotary Encoder Menu Selector

Navigate and select items in a custom LCD screen menu system using a rotary encoder on the VEGA ARIES v3 board.

## Goal
Learn how to read and decode quadrature signals (CLK and DT) from a rotary encoder, handle button click switches (SW), and update a dynamic I2C LCD menu list.

## What You Will Build
An interactive user menu console. Turning the rotary encoder dial shifts the highlighted arrow (`>`) between three menu items (`Option 1`, `Option 2`, `Option 3`) on the bottom row of the LCD screen. Pressing the encoder's shaft button confirms the selection.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Rotary Encoder (KY-040) | `rotary_encoder` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Rotary Encoder | +5V | 5V | Red | Encoder operating voltage (5V) |
| Rotary Encoder | GND | GND | Black | Ground reference |
| Rotary Encoder | CLK | GPIO 14 | Yellow | Quadrature Clock output |
| Rotary Encoder | DT | GPIO 15 | Green | Quadrature Data output |
| Rotary Encoder | SW | GPIO 12 | Blue | Push button switch output |
| I2C LCD | VCC | 5V | Red | LCD power supply (5V) |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | SDA0 (GP17) | Blue | I2C Data line |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** A rotary encoder has an internal switch SW. In the code, set SW to use an internal pull-up (`INPUT_PULLUP`) so it returns `LOW` when pressed and `HIGH` when released.

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int CLK_PIN = 14; // CLK on GPIO 14
const int DT_PIN = 15;  // DT on GPIO 15
const int SW_PIN = 12;  // SW on GPIO 12

LiquidCrystal_I2C lcd(0x27, 16, 2);

int lastClkState = -1;
int menuIndex = 0;
const int MAX_MENU_ITEMS = 3; 
int lastMenuIndex = -1;
int lastSwState = -1;

void setup() {
  Wire.begin();
  
  pinMode(CLK_PIN, INPUT);
  pinMode(DT_PIN, INPUT);
  pinMode(SW_PIN, INPUT_PULLUP);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Select Mode:");

  // Read starting clock status
  lastClkState = digitalRead(CLK_PIN);
}

void loop() {
  int clkState = digitalRead(CLK_PIN);
  int swState = digitalRead(SW_PIN);

  // Quadrature State Machine check (runs without loops)
  if (clkState != lastClkState) {
    if (clkState == LOW) { // Read rising edge transitions
      if (digitalRead(DT_PIN) == HIGH) {
        menuIndex++;
        if (menuIndex >= MAX_MENU_ITEMS) {
          menuIndex = 0; // Wrap around
        }
      } else {
        menuIndex--;
        if (menuIndex < 0) {
          menuIndex = MAX_MENU_ITEMS - 1; // Wrap around
        }
      }
    }
    lastClkState = clkState;
  }

  // Update LCD text if menu index has changed
  if (menuIndex != lastMenuIndex) {
    lcd.setCursor(0, 1);
    if (menuIndex == 0) {
      lcd.print("> Option A      ");
    } else if (menuIndex == 1) {
      lcd.print("> Option B      ");
    } else if (menuIndex == 2) {
      lcd.print("> Option C      ");
    }
    lastMenuIndex = menuIndex;
  }

  // Confirm selection on button click
  if (swState != lastSwState) {
    if (swState == LOW) {
      lcd.setCursor(0, 0);
      lcd.print("Selected:       ");
      lcd.setCursor(10, 0);
      lcd.print(menuIndex + 1);
      
      delay(600); // Visual pause for selection confirmation
      
      lcd.setCursor(0, 0);
      lcd.print("Select Mode:    ");
    }
    lastSwState = swState;
  }

  delay(5); // Sampling interval / debouncing
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **Rotary Encoder**, and **I2C LCD Display** components onto the canvas.
2. Wire the Encoder: **+5V** to **5V**, **GND** to **GND**, **CLK** to **GPIO 14**, **DT** to **GPIO 15**, and **SW** to **GPIO 12**.
3. Wire the LCD: **VCC** to **5V**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Encoder rotated. MenuIndex: 1
Encoder rotated. MenuIndex: 2
Button Pressed! Confirmed selection: Option C
```

## Expected Canvas Behavior
* Rotating the simulated encoder knob clockwise moves the highlighted cursor arrow from Option A to Option B to Option C.
* Rotating counter-clockwise shifts the cursor arrow in the reverse direction.
* Clicking the encoder switch updates the top row of the LCD to display the selected option number for a brief moment before returning to `Select Mode:`.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(SW_PIN, INPUT_PULLUP)` | Activates internal pull-up on the encoder push-button input. |
| `digitalRead(CLK_PIN)` | Samples the current status of the CLK line. |
| `clkState != lastClkState` | Monitors transitions on the encoder wheel. |
| `digitalRead(DT_PIN) == HIGH` | Evaluates phase differences to determine clockwise vs counter-clockwise rotation. |
| `lcd.print("> Option A      ")` | Prints option labels with trailing spaces to erase old characters. |

## Hardware & Safety Concept
* **Quadrature Encoding Theory**: A rotary encoder features two internal metal contacts that slide across a segmented wheel. The physical spacing of the contact pads generates two square waves shifted 90 degrees out of phase (quadrature). By identifying which pin changes state first, the microcontroller determines the rotation direction.
* **Switch Bounce Filtering**: Mechanical switches bounce on contact, generating transient noise spikes that can be misread as rapid duplicate button presses. We use a short `delay(5)` sampling filter in the loop to debounce the signals.

## Try This! (Challenges)
1. **Dynamic Scroll Limit**: Connect a warning LED to GPIO 15. If the user tries to turn the encoder past the boundaries (rather than wrapping around), turn on the LED for 200 ms to indicate a limit hit.
2. **Speed Scaling counter**: Modify the encoder logic to adjust a numerical setting (0 to 100). If the encoder is rotated rapidly, increase the counter increment step size from 1 to 5.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Menu item changes back and forth erratically | CLK and DT pins crossed or loose | Swap the CLK and DT wires in the wiring layout or reverse their digital pin definitions in code. |
| Button click does not confirm selection | Button pin not pulled up | Ensure `SW_PIN` is configured as `INPUT_PULLUP` and wired properly to GPIO 12. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [58 - 16x2 I2C LCD Hello World](58-16x2-i2c-lcd-hello-world.md)
- [115 - Keypad Display HUD](115-keypad-display-hud.md)
