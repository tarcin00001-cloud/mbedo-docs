# 197 - ESP32 Dual Axis Robotic Arm PID Position Controller

Build a closed-loop robotic arm joint controller on the ESP32 that reads target angles from two input potentiometers, measures actual joint angles using two feedback potentiometers on the joint axes, and runs two independent PID loops to drive the base and shoulder servos to match the targets.

## Goal
Learn how to implement dual closed-loop PID control loops for robotic joints, read analog feedback sensors, and minimize positional error.

## What You Will Build
Two potentiometers on GPIO 34 and 35 set target joint angles. Two feedback potentiometers on GPIO 32 and 33 measure actual joint angles. Two servos on GPIO 12 and 13 steer the arm. The ESP32 runs two independent PID control loops to calculate corrections, adjusting the servos to align the joint angles with the targets.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometers (4) | `potentiometer` | Yes | Yes |
| Servo Motors (2) | `servo` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| External Power Supply (5V, 2A) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Target Base Pot | Wiper | GPIO34 | Yellow | Base target input |
| Target Shoulder Pot| Wiper | GPIO35 | Yellow | Shoulder target input |
| Feedback Base Pot | Wiper | GPIO32 | White | Base actual feedback |
| Feedback Shoulder Pot| Wiper | GPIO33 | White | Shoulder actual feedback |
| Servo Base | Signal | GPIO12 | Orange | Base rotation servo |
| Servo Shoulder | Signal | GPIO13 | Blue | Shoulder joint servo |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the common ground for all potentiometers. Power the servos from the 5V Vin rail to prevent voltage drops.

## Code
```cpp
// Dual Axis Robotic Arm PID Position Controller (Closed-loop Servos)
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <ESP32Servo.h>

// Pin Definitions
const int PIN_TGT_BASE = 34; // Pot 1
const int PIN_TGT_SHLD = 35; // Pot 2
const int PIN_FDB_BASE = 32; // Pot 3
const int PIN_FDB_SHLD = 33; // Pot 4

const int SERVO_BASE_PIN = 12;
const int SERVO_SHLD_PIN = 13;

LiquidCrystal_I2C lcd(0x27, 16, 2);
Servo servoBase;
Servo servoShoulder;

// PID Tuning Constants
const float KP = 1.2;
const float KI = 0.1;
const float KD = 0.05;

// Servos safety range limits (prevents mechanical binding)
const int SERVO_MIN = 10;
const int SERVO_MAX = 170;

// PID State structure
struct JointPID {
  float error = 0.0;
  float lastError = 0.0;
  float integral = 0.0;
  float derivative = 0.0;
  
  float compute(float target, float actual, float dt) {
    error = target - actual;
    
    // Integral with windup clamping
    integral += error * dt;
    integral = constrain(integral, -30, 30);
    
    // Derivative
    derivative = (error - lastError) / dt;
    lastError = error;
    
    return (KP * error) + (KI * integral) + (KD * derivative);
  }
};

JointPID pidBase;
JointPID pidShoulder;

unsigned long lastTime = 0;
const unsigned long UPDATE_INTERVAL_MS = 25; // 40Hz update loop

void setup() {
  Serial.begin(115200);
  
  servoBase.attach(SERVO_BASE_PIN);
  servoShoulder.attach(SERVO_SHLD_PIN);
  
  servoBase.write(90);
  servoShoulder.write(90);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Closed-Loop Arm");
  lcd.setCursor(0, 1);
  lcd.print("PID Init...");
  
  delay(1500);
  lcd.clear();
  
  lastTime = millis();
}

void loop() {
  unsigned long now = millis();
  
  if (now - lastTime >= UPDATE_INTERVAL_MS) {
    float dt = (now - lastTime) / 1000.0;
    lastTime = now;
    
    // 1. Read Target Angles (Scale 0-4095 to 0-180 degrees)
    float targetBase = map(analogRead(PIN_TGT_BASE), 0, 4095, 0, 180);
    float targetShoulder = map(analogRead(PIN_TGT_SHLD), 0, 4095, 0, 180);
    
    // 2. Read Feedback Angles (Scale 0-4095 to 0-180 degrees)
    float actualBase = map(analogRead(PIN_FDB_BASE), 0, 4095, 0, 180);
    float actualShoulder = map(analogRead(PIN_FDB_SHLD), 0, 4095, 0, 180);
    
    // 3. Compute PID corrections
    float corrBase = pidBase.compute(targetBase, actualBase, dt);
    float corrShoulder = pidShoulder.compute(targetShoulder, actualShoulder, dt);
    
    // Calculate new servo drive positions (Base center + correction)
    int posBase = 90 + (int)corrBase;
    int posShoulder = 90 + (int)corrShoulder;
    
    // Constrain outputs to safety limits
    posBase = constrain(posBase, SERVO_MIN, SERVO_MAX);
    posShoulder = constrain(posShoulder, SERVO_MIN, SERVO_MAX);
    
    // 4. Command Servos
    servoBase.write(posBase);
    servoShoulder.write(posShoulder);
    
    // 5. Update LCD HUD (switch displays periodically)
    static int cycleCounter = 0;
    cycleCounter = (cycleCounter + 1) % 8; // Update LCD less frequently
    
    if (cycleCounter == 0) {
      lcd.setCursor(0, 0);
      lcd.print("B_Tg:"); lcd.print((int)targetBase);
      lcd.print(" B_Ac:"); lcd.print((int)actualBase);
      lcd.print("   ");
      
      lcd.setCursor(0, 1);
      lcd.print("S_Tg:"); lcd.print((int)targetShoulder);
      lcd.print(" S_Ac:"); lcd.print((int)actualShoulder);
      lcd.print("   ");
      
      Serial.print("Base Error: "); Serial.print(pidBase.error, 1);
      Serial.print(" | Shoulder Error: "); Serial.println(pidShoulder.error, 1);
    }
  }
  
  delay(5);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, four **Potentiometers**, two **Servos**, and **16x2 I2C LCD** onto the canvas.
2. Wire inputs, Servo Base to **GPIO12**, Servo Shoulder to **GPIO13**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide Target Base Pot (GPIO 34) right (setting target to 120°). The Base servo rotates.
5. Slide the corresponding Feedback Base Pot (GPIO 32) to 120°. The servo stops moving once the target is aligned.

## Expected Output
Serial Monitor:
```
Base Error: 30.0 | Shoulder Error: 0.0
Base Error: 15.2 | Shoulder Error: 0.0
Base Error: 0.2 | Shoulder Error: 0.0
```

LCD Display:
```
B_Tg:120 B_Ac:120
S_Tg:90  S_Ac:90
```

## Expected Canvas Behavior
* The LCD displays the target and feedback angles.
* Adjusting the Target Base Pot slider rotates the Base servo widget.
* Adjusting the corresponding Feedback Base Pot slider reduces the servo rotation speed, stopping it once the feedback matches the target.
* The system behaves as a closed-loop control system.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pidBase.compute(...)` | Calculates the PID correction based on the difference between target and feedback angles. |
| `90 + (int)corrBase` | Adjusts the center point bias of the servo based on the calculated correction. |
| `constrain(...)` | Limits servo travel to prevent mechanical strain or wire binding. |

## Hardware & Safety Concept: Closed-Loop Position Controllers in Robotics
Traditional hobby servo motors are **open-loop** relative to the microcontroller: the controller sends a PWM command (e.g. 90°), but does not verify if the arm actually moved. If the arm hits an obstacle or carries a heavy load, it can stall or fail to reach the target. Industrial robotic arms use **closed-loop position controllers**: they measure actual joint angles using encoders or potentiometers, and run PID loops to adjust the motors, ensuring high accuracy.

## Try This! (Challenges)
1. **Interactive Tuning Console**: Allow the user to input custom PID parameters (`KP`, `KI`, `KD`) via the Serial Monitor at runtime.
2. **Positional Lock Alert**: Sound a buzzer (GPIO 15) if a joint's error exceeds 20 degrees for more than 2 seconds, indicating a mechanical jam.
3. **Sequence Recording memory**: Add a button on GPIO 4 to record joint position steps, saving the sequence to EEPROM (Project 182).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servos vibrate and shake wildly | PID gain parameters too high | Reduce `KP` and `KD` constants by half |
| Servos rotate to limits and stall | Feedback loop direction inverted | Swap the calculation sign in the servo write calls (e.g. change `90 + corr` to `90 - corr`) |
| LCD display freezes | Mutex or memory deadlock | Check that the I2C bus connections are stable and verify the address |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [133 - ESP32 2-axis Robotic Arm Joint Controller](../intermediate/133-esp32-2-axis-robotic-arm-joint-controller.md)
- [152 - ESP32 Servo PID Angle Balancer](152-esp32-servo-pid-angle-balancer.md)
- [182 - ESP32 EEPROM Settings Store & Restore](182-esp32-eeprom-settings-store-and-restore.md)
