# 133 - ESP32 2-Axis Robotic Arm Joint Controller

Build a 2-axis manual joysticker or control console that maps two potentiometer inputs to drive Base and Shoulder servos of a robotic arm, displaying live angles on a 16x2 LCD.

## Goal
Learn how to read and filter multiple analog inputs simultaneously, manage multiple servo instances using `ESP32Servo`, and display multi-row coordinate labels on LCDs.

## What You Will Build
Two potentiometers are connected to GPIO 34 and 35. Two servos (Base on GPIO 13, Shoulder on GPIO 12) are mapped to these inputs. A 16x2 I2C LCD displays the joint angles: Row 1 displays the Base angle and Row 2 displays the Shoulder angle.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometers (2) | `potentiometer` | Yes | Yes |
| Servos (2) | `servo` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer 1 (Base) | Wiper | GPIO34 | Yellow | Base joint control |
| Potentiometer 1 (Base) | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| Potentiometer 2 (Shoulder)| Wiper | GPIO35 | Yellow | Shoulder joint control |
| Potentiometer 2 (Shoulder)| VCC / GND | 3V3 / GND | Red / Black | Power rails |
| Servo 1 (Base) | Signal | GPIO13 | Orange | Base servo PWM |
| Servo 1 (Base) | VCC / GND | 5V / GND | Red / Black | Power rails |
| Servo 2 (Shoulder) | Signal | GPIO12 | Blue | Shoulder servo PWM |
| Servo 2 (Shoulder) | VCC / GND | 5V / GND | Red / Black | Power rails |
| I2C LCD | VCC / GND | 5V / GND | Red / Black | LCD power |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |

> **Wiring tip:** When controlling multiple servos, power them from an external 5V power supply. Do not power multiple servos from the ESP32's 3.3V or 5V regulator, as this can cause the ESP32 to brown out and reset. Ensure all grounds are tied together.

## Code
```cpp
// 2-Axis Robotic Arm Joint Controller
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <ESP32Servo.h>

const int POT_X = 34; // Base Potentiometer
const int POT_Y = 35; // Shoulder Potentiometer

const int SERVO_X = 13; // Base Servo
const int SERVO_Y = 12; // Shoulder Servo

Servo baseServo;
Servo shoulderServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Simple filtering structure for each analog channel
struct AnalogFilter {
  static const int SIZE = 8;
  int buffer[SIZE];
  int index = 0;
  long sum = 0;
  
  void init(int startVal) {
    for (int i = 0; i < SIZE; i++) buffer[i] = startVal;
    sum = (long)startVal * SIZE;
  }
  
  int filter(int val) {
    sum -= buffer[index];
    buffer[index] = val;
    sum += val;
    index = (index + 1) % SIZE;
    return sum / SIZE;
  }
};

AnalogFilter filterX;
AnalogFilter filterY;

void setup() {
  Serial.begin(115200);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("2-Axis Controller");
  lcd.setCursor(0, 1);
  lcd.print("Active...");
  
  // Initialize filter buffers
  filterX.init(analogRead(POT_X));
  filterY.init(analogRead(POT_Y));
  
  baseServo.attach(SERVO_X);
  shoulderServo.attach(SERVO_Y);
  
  baseServo.write(90);
  shoulderServo.write(90);
  
  delay(1000);
  lcd.clear();
}

void loop() {
  // Read raw values
  int rawX = analogRead(POT_X);
  int rawY = analogRead(POT_Y);
  
  // Filter values
  int smoothX = filterX.filter(rawX);
  int smoothY = filterY.filter(rawY);
  
  // Map values to servo angles
  int angleX = map(smoothX, 0, 4095, 0, 180);
  int angleY = map(smoothY, 0, 4095, 0, 180);
  
  // Command servos
  baseServo.write(angleX);
  shoulderServo.write(angleY);
  
  // Update LCD Display
  lcd.setCursor(0, 0);
  lcd.print("Base: ");
  lcd.print(angleX);
  lcd.print((char)223);
  lcd.print("        "); // Clear line end
  
  lcd.setCursor(0, 1);
  lcd.print("Shoulder: ");
  lcd.print(angleY);
  lcd.print((char)223);
  lcd.print("        ");
  
  Serial.print("Base: "); Serial.print(angleX);
  Serial.print(" | Shoulder: "); Serial.println(angleY);
  
  delay(50); // 20Hz update rate
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, two **Potentiometers**, two **Servos**, and **16x2 I2C LCD** onto the canvas.
2. Wire Pot 1 to **GPIO34**, Pot 2 to **GPIO35**, Servo 1 to **GPIO13**, Servo 2 to **GPIO12**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide Pot 1 on the canvas. Watch Servo 1 rotate. Slide Pot 2 and watch Servo 2 rotate. Both angles are shown on the LCD.

## Expected Output
Serial Monitor:
```
Base: 90 | Shoulder: 90
Base: 120 | Shoulder: 45
Base: 180 | Shoulder: 0
```

LCD Display:
```
Base: 120°
Shoulder: 45°
```

## Expected Canvas Behavior
* Sliding Pot 1 slider adjusts the Base servo widget position.
* Sliding Pot 2 slider adjusts the Shoulder servo widget position.
* The LCD updates both values independently in real time.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `AnalogFilter filterX` | Instantiates a separate filtering helper struct for the Base potentiometer. |
| `baseServo.write(angleX)` | Updates the PWM output for the first servo. |
| `shoulderServo.write(angleY)` | Updates the PWM output for the second servo. |

## Hardware & Safety Concept: Current Draws in Dual Servo Configurations
Servos draw significant current peaks when starting or carrying a load. Each active SG90 servo can draw up to **500 mA** under load. Running two servos simultaneously can draw up to **1.0 A** of current, which exceeds the current limit of the ESP32's onboard regulator. To prevent brownout resets, always power the servos from an external 5V regulator, keeping the logic ground shared.

## Try This! (Challenges)
1. **Synchronized Mirror Mode**: Add a slide switch on GPIO 4. When flipped, make Pot 1 control both servos in opposite directions (mirror mode).
2. **Speed limiting sweeps**: Prevent the servos from moving faster than 30 degrees per second by implementing step-based ramping.
3. **Safety limits warning**: Sound a buzzer (GPIO 15) if either servo is driven past 160° or below 20°.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Microcontroller resets when sliding both knobs | Power supply overload (brownout) | Connect the servos to an external 5V power supply rather than the ESP32's 5V pin |
| Servo movements are swapped | Pin definitions swapped | Swap the `SERVO_X` and `SERVO_Y` pin numbers in the code |
| Jittering servos | Analog noise on ADC lines | Ensure VCC connections are stable; increase the filter buffer size |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [132 - ESP32 Servo Robotic Arm Base Rotation](132-esp32-servo-robotic-arm-base-rotation.md)
- [134 - ESP32 3-Axis Robotic Arm Claw Controller](134-esp32-3-axis-robotic-arm-claw-controller.md)
- [52 - ESP32 Dual Servo Coordinates Sweep](../intermediate/52-esp32-dual-servo-coordinates-sweep.md)
