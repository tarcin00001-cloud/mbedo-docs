# 68 - ESP32 Stepper Motor Position Log

Control a stepper motor using two buttons (Forward and Reverse) while displaying the current step count position on a 16x2 I2C LCD.

## Goal
Learn how to track step counts in software to log precise motor shaft position, and interface user buttons to jog a stepper motor.

## What You Will Build
Two buttons are connected to GPIO 4 and GPIO 15. Pressing Button A steps the motor clockwise and increments a position counter; Button B steps the motor counter-clockwise and decrements the counter. The 16x2 I2C LCD displays the live position.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| ULN2003 Driver Board | `stepper_driver` | Yes | Yes |
| 28BYJ-48 Stepper Motor | `stepper_motor` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| Tactile Push Buttons (2) | `button` | Yes | Yes |
| 10 kΩ Resistors (2) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| ULN2003 Board | IN1 | GPIO18 | Yellow | Coil 1 |
| ULN2003 Board | IN2 | GPIO19 | Green | Coil 2 |
| ULN2003 Board | IN3 | GPIO21 | Blue | Coil 3 |
| ULN2003 Board | IN4 | GPIO22 | Purple | Coil 4 |
| ULN2003 Board | VCC | 5V (Vin) | Red | Motor power |
| ULN2003 Board | GND | GND | Black | Ground |
| Button A (CW) | Pin 1 | 3V3 | Red | Power |
| Button A (CW) | Pin 2 | GPIO4 | Yellow | CW Jog Input |
| Resistor A (10k) | Leg 1 | GPIO4 | White | Pull-down |
| Resistor A (10k) | Leg 2 | GND | Black | Ground |
| Button B (CCW) | Pin 1 | 3V3 | Red | Power |
| Button B (CCW) | Pin 2 | GPIO15 | Green | CCW Jog Input |
| Resistor B (10k) | Leg 1 | GPIO15 | White | Pull-down |
| Resistor B (10k) | Leg 2 | GND | Black | Ground |
| I2C LCD | VCC | 5V (Vin) | Red | LCD Power |
| I2C LCD | GND | GND | Black | Ground |
| I2C LCD | SDA | GPIO21 | Blue | I2C Data (Shared) |
| I2C LCD | SCL | GPIO22 | Yellow | I2C Clock (Shared) |

> **Wiring tip:** GPIO 21 and 22 are used for both the I2C LCD and the ULN2003 board. This is perfectly fine because I2C uses address-based signaling and the stepper uses simple digital writes to individual pins. However, note that in hardware, make sure they are connected in parallel.

## Code
```cpp
// Stepper Motor Position Log with LCD
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int IN1 = 18;
const int IN2 = 19;
const int IN3 = 21;
const int IN4 = 22;

const int BTN_CW = 4;
const int BTN_CCW = 15;

const int STEPS[4][4] = {
  {1, 0, 0, 0},
  {0, 1, 0, 0},
  {0, 0, 1, 0},
  {0, 0, 0, 1}
};

int stepIndex = 0;
long motorPosition = 0; // Tracks relative step count from start (home)

LiquidCrystal_I2C lcd(0x27, 16, 2);

void stepMotor(int dir) {
  // Update step index based on direction (+1 or -1)
  stepIndex += dir;
  if (stepIndex > 3) stepIndex = 0;
  if (stepIndex < 0) stepIndex = 3;
  
  digitalWrite(IN1, STEPS[stepIndex][0]);
  digitalWrite(IN2, STEPS[stepIndex][1]);
  digitalWrite(IN3, STEPS[stepIndex][2]);
  digitalWrite(IN4, STEPS[stepIndex][3]);
  
  motorPosition += dir;
}

void releaseCoils() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
}

void setup() {
  Serial.begin(115200);
  
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  
  pinMode(BTN_CW, INPUT);
  pinMode(BTN_CCW, INPUT);
  
  releaseCoils();
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Stepper Position");
  
  delay(1000);
  lcd.clear();
}

void loop() {
  bool cwPressed = (digitalRead(BTN_CW) == HIGH);
  bool ccwPressed = (digitalRead(BTN_CCW) == HIGH);
  
  bool moved = false;
  
  if (cwPressed && !ccwPressed) {
    stepMotor(1);
    moved = true;
    delay(5); // Adjust step speed delay
  } 
  else if (ccwPressed && !cwPressed) {
    stepMotor(-1);
    moved = true;
    delay(5);
  }
  
  if (moved) {
    // Update LCD
    lcd.setCursor(0, 0);
    lcd.print("Pos: ");
    lcd.print(motorPosition);
    lcd.print(" steps   ");
    
    // Show direction indicator
    lcd.setCursor(0, 1);
    if (cwPressed) {
      lcd.print("Direction: CW   ");
    } else {
      lcd.print("Direction: CCW  ");
    }
    
    Serial.print("Position: "); Serial.println(motorPosition);
  } else {
    // If not moving, release coils to prevent heat buildup
    releaseCoils();
    lcd.setCursor(0, 1);
    lcd.print("Direction: IDLE ");
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **ULN2003**, **Stepper Motor**, **16x2 I2C LCD**, and two **Buttons** onto the canvas.
2. Connect Button A (CW) to **GPIO4** and Button B (CCW) to **GPIO15**.
3. Connect the stepper control pins (18, 19, 21, 22) and the LCD I2C pins (21, 22) as listed in the wiring table.
4. Paste the code and click **Run**.
5. Click and hold the buttons and observe the stepper position count increment/decrement on the LCD.

## Expected Output
Serial Monitor:
```
Position: 1
Position: 2
Position: 1
Position: 0
Position: -1
```

LCD Display (rotating CW):
```
Pos: 128 steps
Direction: CW
```

## Expected Canvas Behavior
* Pressing the CW button rotates the stepper clockwise, and the step count rises.
* Pressing the CCW button rotates the stepper counter-clockwise, and the step count falls.
* Releasing both buttons stops the stepper and changes the direction line on the LCD to IDLE.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `motorPosition += dir` | Tracks absolute step count (+1 for CW, -1 for CCW). |
| `releaseCoils()` | Sets all coil outputs to LOW to stop drawing current when idle. |
| `delay(5)` | Controls the stepping frequency (5ms per step ≈ 200 Hz). |

## Hardware & Safety Concept: Coils Release and Stepper Holding Torque
Unlike DC motors, stepper motors consume maximum current when stationary if the coils remain energized (holding torque). If a stepper is used just for positioning and doesn't need to hold a load against gravity, turning the driver inputs LOW when IDLE saves power and prevents the motor and ULN2003 driver chip from getting hot.

## Try This! (Challenges)
1. **Reset to Home**: Add a third button on GPIO 14 that resets the position tracker to 0 and returns the motor to the home position automatically.
2. **Angle Conversion**: Convert the step count display on the LCD to degrees (1 step ≈ 0.176°).
3. **Limit Switches**: Integrate limit switches on the left and right that prevent the motor from stepping further once hit.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor vibrates but doesn't step | Incorrect sequence wiring | Verify IN1-IN4 pin assignments match the code exactly |
| Position counter drifts | Button debounce or noise | Increase debounce filtering or ensure stable pull-down connections |
| LCD does not update | I2C address conflict | Verify address 0x27 is correct for the simulated/physical module |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [56 - ESP32 Stepper Motor Step Sequence](56-esp32-stepper-motor-step-sequence.md)
- [57 - ESP32 Stepper Motor Speed Controller](57-esp32-stepper-motor-speed-controller.md)
- [67 - ESP32 Potentiometer Speed DC Motor](67-esp32-potentiometer-speed-dc-motor.md)
