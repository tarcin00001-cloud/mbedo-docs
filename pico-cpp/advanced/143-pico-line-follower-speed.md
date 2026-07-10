# 143 - Pico Line Follower Speed

Build an autonomous line-following robot that dynamically scales its wheel speeds for smooth turn correction and displays status on an LCD.

## Goal
Learn how to implement differential steering with adjustable PWM speed control and log path states on I2C screen indicators.

## What You Will Build
A smooth autonomous line tracker:
- **Left/Right IR Sensors (GP16, GP17)**: Detect track line edge transitions.
- **L298N & Dual DC Motors (GP10-15)**: Automatically steers the robot by running one wheel at full speed (255) and the other at half speed (120) when correcting, rather than stopping the wheel completely, creating smoother turns.
- **16x2 I2C LCD (GP4, GP5)**: Displays active steering states.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| IR Reflective Sensors | `button` | Yes (represented by two switches) | Yes (TCRT5000 modules) |
| L298N Motor Driver | `l298n` | Yes | Yes |
| DC Motors | `dc_motor` | Yes (two motors) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Left IR Sensor | OUT | GP16 | Left track monitor |
| Right IR Sensor | OUT | GP17 | Right track monitor |
| L298N Control | ENA / ENB | GP10 / GP11 | Left/Right Speed PWM |
| L298N Control | IN1 / IN2 | GP12 / GP13 | Left wheel direction |
| L298N Control | IN3 / IN4 | GP14 / GP15 | Right wheel direction |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int LEFT_IR  = 16;
const int RIGHT_IR = 17;

const int ENA = 10; const int ENB = 11;
const int IN1 = 12; const int IN2 = 13;
const int IN3 = 14; const int IN4 = 15;

LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(LEFT_IR, INPUT);
  pinMode(RIGHT_IR, INPUT);

  pinMode(ENA, OUTPUT); pinMode(ENB, OUTPUT);
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Smart Tracker");
  lcd.setCursor(0, 1);
  lcd.print("Engine Online   ");
  delay(1000);
}

void loop() {
  int leftState  = digitalRead(LEFT_IR);
  int rightState = digitalRead(RIGHT_IR);

  lcd.clear();
  lcd.setCursor(0, 0);

  // 1. Both on white floor: move forward at standard speed (180 out of 255)
  if (leftState == LOW && rightState == LOW) {
    analogWrite(ENA, 180);
    analogWrite(ENB, 180);
    moveForward();
    lcd.print("Path: STRAIGHT  ");
  }
  // 2. Left sensor hits line: drift right wheel forward, slow left wheel down
  else if (leftState == HIGH && rightState == LOW) {
    analogWrite(ENA, 80);  // Slow left
    analogWrite(ENB, 220); // Fast right
    moveForward();
    lcd.print("Path: ADJUST L  ");
  }
  // 3. Right sensor hits line: drift left wheel forward, slow right wheel down
  else if (leftState == LOW && rightState == HIGH) {
    analogWrite(ENA, 220); // Fast left
    analogWrite(ENB, 80);  // Slow right
    moveForward();
    lcd.print("Path: ADJUST R  ");
  }
  // 4. Both sensors hit line: stop robot
  else if (leftState == HIGH && rightState == HIGH) {
    analogWrite(ENA, 0);
    analogWrite(ENB, 0);
    stopRobot();
    lcd.print("Path: BLOCKED   ");
  }

  delay(50);
}

void moveForward() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
}

void stopRobot() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);  digitalWrite(IN4, LOW);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two IR sensors** (represented by switch buttons), **L298N**, **two DC Motors**, and **I2C LCD** onto the canvas.
2. Connect IR sensors to **GP16/GP17**, L298N to **GP10-15**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Toggle the sensor switches and observe the wheel speed scaling and LCD feedback.

## Expected Output

Terminal:
```
Simulation active. Speed scaling line tracker active.
```

## Expected Canvas Behavior
* Both sensors LOW (straight): Both wheels spin forward at moderate speed (180). LCD reads `Path: STRAIGHT`.
* Left sensor HIGH (adjusting): Left wheel slows down (80), right wheel speeds up (220) to steer back onto the track. LCD reads `Path: ADJUST L`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `analogWrite(ENA, 80)` | Slows down the inner wheel during a turn to create a smooth, curved path correction rather than a sharp pivot. |

## Hardware & Safety Concept: Smooth Differential Steering
Stopping one motor completely to make a turn (hard pivots) causes line-following robots to jerk and shake, often overcorrecting and losing the track line at high speeds. Using proportional PWM speed control keeps both wheels spinning at different rates, creating a smooth, curved path correction that keeps the robot stable.

## Try This! (Challenges)
1. **Warning Indicator**: Connect a buzzer on GP14 and beep if the robot stops at a blocked line.
2. **Dynamic Base Speed**: Connect a potentiometer on GP26 to adjust the robot's base speed dynamically.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot turns in circles on straight paths | Motor wiring reversed | Swap the direction wire inputs (IN1/IN2 or IN3/IN4) of the motor that is spinning backward, or swap the pin definitions in code. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [112 - Pico Bluetooth Robot](../intermediate/112-pico-bt-robot.md)
- [113 - Pico Line Follower](../intermediate/113-pico-line-follower.md)
