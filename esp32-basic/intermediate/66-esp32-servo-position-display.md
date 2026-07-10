# 66 - ESP32 Servo Position Display

Read a potentiometer to control a servo motor's angle while displaying the live target angle on a 16x2 I2C LCD.

## Goal
Learn how to integrate analog input, servo PWM control, and I2C LCD character display to build an interactive, closed-loop style feedback interface.

## What You Will Build
A potentiometer controls the servo angle (0° to 180°). The 16x2 I2C LCD displays the target angle in degrees along with a visual progress indicator representing the position.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Left leg | 3V3 | Red | Power rail |
| Potentiometer | Right leg | GND | Black | Ground rail |
| Potentiometer | Wiper | GPIO34 | Yellow | ADC Input |
| Servo Motor | Signal | GPIO5 | Orange | PWM control |
| Servo Motor | VCC | 5V (Vin) | Red | Motor power |
| Servo Motor | GND | GND | Black | Ground return |
| I2C LCD | VCC | 5V (Vin) | Red | LCD power |
| I2C LCD | GND | GND | Black | Ground return |
| I2C LCD | SDA | GPIO21 | Blue | I2C Data |
| I2C LCD | SCL | GPIO22 | Yellow | I2C Clock |

> **Wiring tip:** Both the servo motor and the I2C LCD require 5V to run correctly. You can share the Vin pin of the ESP32 to power both. Connect all ground wires to the common GND rail to ensure consistent references.

## Code
```cpp
// Servo Position display
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <ESP32Servo.h>

const int POT_PIN = 34;
const int SERVO_PIN = 5;

LiquidCrystal_I2C lcd(0x27, 16, 2);
Servo myServo;

void setup() {
  Serial.begin(115200);
  
  // Initialise LCD
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Servo Controller");
  
  // Attach Servo
  myServo.attach(SERVO_PIN);
  
  delay(1000);
  lcd.clear();
}

void loop() {
  int raw = analogRead(POT_PIN);
  int angle = map(raw, 0, 4095, 0, 180);
  angle = constrain(angle, 0, 180);
  
  myServo.write(angle);
  
  // Print to LCD
  lcd.setCursor(0, 0);
  lcd.print("Angle: ");
  lcd.print(angle);
  lcd.print(" deg   "); // Trailing spaces clear old digits
  
  // Draw a simple horizontal bar representing angle
  lcd.setCursor(0, 1);
  int numBars = map(angle, 0, 180, 0, 16);
  for (int i = 0; i < 16; i++) {
    if (i < numBars) {
      lcd.print("=");
    } else {
      lcd.print(" ");
    }
  }
  
  Serial.print("Pot ADC: "); Serial.print(raw);
  Serial.print(" | Angle: "); Serial.println(angle);
  
  delay(50);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer**, **Servo**, and **16x2 I2C LCD** onto the canvas.
2. Wire Potentiometer wiper to **GPIO34**, Servo signal to **GPIO5**, LCD SDA to **GPIO21**, and SCL to **GPIO22**.
3. Paste the code and click **Run**.
4. Adjust the potentiometer slider and observe the LCD updating alongside the servo rotation.

## Expected Output
Serial Monitor:
```
Pot ADC: 2048 | Angle: 90
Pot ADC: 4095 | Angle: 180
Pot ADC: 0 | Angle: 0
```

LCD Display (at 90 degrees):
```
Angle: 90 deg
========
```

## Expected Canvas Behavior
* The servo rotates in sync with the potentiometer widget's position.
* The LCD displays the angle value dynamically.
* The second line of the LCD displays a growing/shrinking bar of "=" characters corresponding to the angle.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `myServo.attach(SERVO_PIN)` | Attaches the Servo object to GPIO 5. |
| `map(raw, 0, 4095, 0, 180)` | Translates the 12-bit ADC reading to a 0–180° range. |
| `lcd.print(" deg   ")` | Appends spaces to prevent old larger numbers from displaying when shifting down (e.g. 100 to 99). |
| `map(angle, 0, 180, 0, 16)` | Scales the angle to the 16 available columns on the LCD row. |

## Hardware & Safety Concept: Current Management with Inductive Loads
Servo motors contain a small DC motor that pulls current spikes during start and stall conditions. Standard USB ports output up to 500mA. Running a servo and backlight LCD together from a USB port can exceed this during high load, leading to voltage brownout (ESP32 resets). To prevent this in physical setups, a decoupling capacitor (e.g. 220 µF) can be placed across the 5V and GND rail near the servo to buffer these current spikes.

## Try This! (Challenges)
1. **Target Arrival Indicator**: Add an LED on GPIO 18 that turns on only when the potentiometer matches the servo target angle (if there is a delayed sweep filter).
2. **Reverse direction**: Reverse the mapping in the code so that turning the potentiometer to the right rotates the servo left.
3. **Calibrated bounds**: Restrict the rotation angle bounds dynamically (e.g., 30° to 150°) using software constraints.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD does not show text | Backlight potentiometer contrast needs adjustment | Turn the small blue potentiometer on the back of the physical LCD module |
| Servo jitters or doesn't move | Unstable power connection | Ensure 5V (Vin) is utilized, not 3V3 |
| LCD shows partial or corrupted digits | Screen buffer not cleared | Add spaces in print statement or use `lcd.clear()` strategically |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [46 - ESP32 Potentiometer Servo Control](../beginner/46-esp32-potentiometer-servo-control.md)
- [51 - ESP32 Servo Motor 180° Sweep](51-esp32-servo-motor-180-sweep.md)
- [67 - ESP32 Potentiometer Speed DC Motor](67-esp32-potentiometer-speed-dc-motor.md)
