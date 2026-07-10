# 111 - ESP32 Rotary Encoder Menu Selector

Build an interactive user interface using a rotary encoder to scroll through menu items on a 16x2 I2C LCD, with push-button select confirmations.

## Goal
Learn how to decode quadrature pulses from a rotary encoder, track rotation direction in software, handle switch bounces, and build a simple menu navigation system.

## What You Will Build
A rotary encoder is connected to GPIO 18 (CLK), GPIO 19 (DT), and GPIO 4 (SW). The 16x2 I2C LCD displays a menu. Turning the encoder scrolls through three options (System Status, Sensor Data, Settings). Clicking the encoder switch selects and executes the option, displaying a confirmation screen.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Rotary Encoder (KY-040) | `rotary_encoder` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Rotary Encoder | CLK | GPIO18 | Yellow | Quadrature clock line |
| Rotary Encoder | DT | GPIO19 | Green | Quadrature data line |
| Rotary Encoder | SW | GPIO4 | Blue | Push-button switch |
| Rotary Encoder | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| I2C LCD | VCC / GND | 5V / GND | Red / Black | LCD power |
| I2C LCD | SDA | GPIO21 | Blue | I2C Data |
| I2C LCD | SCL | GPIO22 | Yellow | I2C Clock |

> **Wiring tip:** Connect CLK, DT, and SW to GPIO 18, 19, and 4. The KY-040 module includes internal pull-up resistors on the board. If using a bare encoder, enable internal pull-up resistors on the inputs using `INPUT_PULLUP`.

## Code
```cpp
// Rotary Encoder Menu Selector
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int CLK_PIN = 18;
const int DT_PIN = 19;
const int SW_PIN = 4;

LiquidCrystal_I2C lcd(0x27, 16, 2);

int menuCounter = 0;
const int MAX_MENU_ITEMS = 3;
const char* menuItems[MAX_MENU_ITEMS] = {
  "1. Sys Status   ",
  "2. Sensor Data  ",
  "3. Settings     "
};

int lastClkState;
bool lastBtnState = HIGH;
unsigned long lastButtonPress = 0;

void setup() {
  Serial.begin(115200);
  
  pinMode(CLK_PIN, INPUT_PULLUP);
  pinMode(DT_PIN, INPUT_PULLUP);
  pinMode(SW_PIN, INPUT_PULLUP);
  
  lcd.init();
  lcd.backlight();
  
  // Read initial CLK state
  lastClkState = digitalRead(CLK_PIN);
  
  updateDisplay();
  Serial.println("Rotary Menu Selector ready.");
}

void loop() {
  int clkState = digitalRead(CLK_PIN);
  
  // Check if encoder has turned
  // If state changed, pulses occurred
  if (clkState != lastClkState && clkState == LOW) {
    // If DT state is different than CLK state, encoder turned CCW
    if (digitalRead(DT_PIN) != clkState) {
      menuCounter--;
      if (menuCounter < 0) menuCounter = MAX_MENU_ITEMS - 1;
    } else {
      // Otherwise encoder turned CW
      menuCounter++;
      if (menuCounter >= MAX_MENU_ITEMS) menuCounter = 0;
    }
    
    Serial.print("Menu Index: "); Serial.println(menuCounter);
    updateDisplay();
  }
  
  lastClkState = clkState;
  
  // Check button press
  bool btnState = digitalRead(SW_PIN);
  if (btnState == LOW && lastBtnState == HIGH) {
    if (millis() - lastButtonPress > 250) { // Debounce
      Serial.print("Selected: "); Serial.println(menuItems[menuCounter]);
      
      // Flash confirmation on LCD
      lcd.setCursor(0, 1);
      lcd.print("> SELECTED <    ");
      delay(800);
      updateDisplay();
      
      lastButtonPress = millis();
    }
  }
  lastBtnState = btnState;
  
  delay(1); // Small loop stabilization delay
}

void updateDisplay() {
  lcd.setCursor(0, 0);
  lcd.print("Select Menu Opt:");
  lcd.setCursor(0, 1);
  lcd.print(menuItems[menuCounter]);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Rotary Encoder**, and **16x2 I2C LCD** onto the canvas.
2. Wire CLK to **GPIO18**, DT to **GPIO19**, SW to **GPIO4**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Click the virtual rotary encoder widget to rotate left/right, and click its center to select. Watch the LCD menu shift.

## Expected Output
Serial Monitor:
```
Rotary Menu Selector ready.
Menu Index: 1
Menu Index: 2
Selected: 3. Settings     
```

LCD Display (scrolling to Option 2):
```
Select Menu Opt:
2. Sensor Data  
```

## Expected Canvas Behavior
* Turning the virtual encoder widget clockwise shifts the displayed menu option on the LCD forward.
* Turning it counter-clockwise scrolls backward.
* Clicking the center of the encoder flashes "> SELECTED <" on the bottom line.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `digitalRead(DT_PIN) != clkState` | Compares the two quadrature outputs to identify rotation direction. |
| `menuCounter++` | Increments index and wraps it around within the array boundaries. |
| `digitalRead(SW_PIN) == LOW` | Detects when the active-LOW rotary switch is pressed. |

## Hardware & Safety Concept: Quadrature Encoder Signals
Rotary encoders output two square waves shifted by 90 degrees (called **quadrature encoding**), labelled channel A (CLK) and channel B (DT). By monitoring which channel changes first (leading phase), the microchip identifies direction: if channel A goes LOW while channel B is HIGH, it turned clockwise; if channel A goes LOW while channel B is LOW, it turned counter-clockwise.

## Try This! (Challenges)
1. **Double row selection HUD**: Display the current selection on Row 1 and the next upcoming menu item on Row 2.
2. **Scroll Volume Controller**: Map the rotary index to adjust the volume/frequency of a buzzer chime (Project 70).
3. **Password Dial**: Build a lock screen where the user must dial in a sequence of three numbers (e.g. 10 - 25 - 5) using the encoder.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Values skip multiple items on a single click | Bounce noise | Add capacitor filters or increase software debounce timing |
| Scroll direction is reversed | CLK and DT pins swapped | Swap GPIO 18 and 19 wires or flip direction conditions in the code |
| Button press does not register | Switch not pulled HIGH | Ensure `INPUT_PULLUP` is declared or add an external 10 kΩ pull-up |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [63 - ESP32 TM1637 Counter Display](63-esp32-tm1637-counter-display.md)
- [58 - ESP32 16×2 I2C LCD Print Text](58-esp32-16x2-i2c-lcd-print-text.md)
- [115 - ESP32 Keypad Display HUD](115-esp32-keypad-display-hud.md)
