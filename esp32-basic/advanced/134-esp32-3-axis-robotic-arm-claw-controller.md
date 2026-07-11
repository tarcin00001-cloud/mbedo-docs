# 134 - ESP32 3-Axis Robotic Arm Claw Controller

Build a 3-axis control station using three potentiometer dials to drive the Base, Shoulder, and Claw (gripper) servos of a robotic arm, displaying coordinates on an OLED display.

## Goal
Learn how to control three independent servo motors, implement custom travel limits for specialized end-effectors (grippers), and format a multi-line OLED status HUD.

## What You Will Build
Three potentiometers are connected to GPIO 34, 35, and 32. Three servos (Base on GPIO 13, Shoulder on GPIO 12, Gripper Claw on GPIO 14) are mapped to these inputs. An SSD1306 OLED displays all joint angles. The claw servo travel is constrained to 10°–90° to prevent grinding the gripper gears.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometers (3) | `potentiometer` | Yes | Yes |
| Servos (3) | `servo` | Yes | Yes |
| 0.96″ SSD1306 OLED (I2C) | `oled` | Yes | Yes |
| External Power Supply (5V, 2A) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Pot 1 (Base) | Wiper | GPIO34 | Yellow | Base joint control |
| Pot 2 (Shoulder) | Wiper | GPIO35 | Yellow | Shoulder joint control |
| Pot 3 (Claw) | Wiper | GPIO32 | Yellow | Claw gripper control |
| Potentiometers (all) | VCC / GND | 3V3 / GND | Red / Black | Shared power rails |
| Servo 1 (Base) | Signal | GPIO13 | Orange | Base PWM |
| Servo 2 (Shoulder) | Signal | GPIO12 | Blue | Shoulder PWM |
| Servo 3 (Claw) | Signal | GPIO14 | Green | Claw PWM |
| Servos (all) | VCC / GND | 5V / GND | Red / Black | Shared power rails |
| SSD1306 OLED | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| SSD1306 OLED | VCC / GND | 3V3 / GND | Red / Black | OLED power rails |

> **Wiring tip:** Standard robotic claws utilize mechanical linkages that jam if driven to 0° or 180°. Pinion gears will strip or slip. Clamp the claw servo software travel boundaries to protect the mechanism.

## Code
```cpp
// 3-Axis Robotic Arm Claw Controller
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <ESP32Servo.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

const int POT_BASE = 34;
const int POT_SHL = 35;
const int POT_CLAW = 32;

const int SERVO_BASE = 13;
const int SERVO_SHL = 12;
const int SERVO_CLAW = 14;

Servo baseServo;
Servo shoulderServo;
Servo clawServo;

// Gripper limits to protect gears
const int CLAW_MIN_ANGLE = 10; // Fully closed
const int CLAW_MAX_ANGLE = 90; // Fully open

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

AnalogFilter fBase, fShl, fClaw;

void setup() {
  Serial.begin(115200);
  
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED allocation failed!");
    while(1) {}
  }
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.println("Arm Init...");
  display.display();
  
  fBase.init(analogRead(POT_BASE));
  fShl.init(analogRead(POT_SHL));
  fClaw.init(analogRead(POT_CLAW));
  
  baseServo.attach(SERVO_BASE);
  shoulderServo.attach(SERVO_SHL);
  clawServo.attach(SERVO_CLAW);
  
  baseServo.write(90);
  shoulderServo.write(90);
  clawServo.write(CLAW_MIN_ANGLE);
  
  delay(1000);
}

void loop() {
  // Read and filter all channels
  int valBase = fBase.filter(analogRead(POT_BASE));
  int valShl  = fShl.filter(analogRead(POT_SHL));
  int valClaw = fClaw.filter(analogRead(POT_CLAW));
  
  // Map angles
  int angleBase = map(valBase, 0, 4095, 0, 180);
  int angleShl  = map(valShl, 0, 4095, 0, 180);
  
  // Constrain claw travel to prevent gear binding
  int angleClaw = map(valClaw, 0, 4095, CLAW_MIN_ANGLE, CLAW_MAX_ANGLE);
  
  // Command Servos
  baseServo.write(angleBase);
  shoulderServo.write(angleShl);
  clawServo.write(angleClaw);
  
  // Update OLED Display HUD
  display.clearDisplay();
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.println("ROBOTIC ARM JOINT HUD");
  display.println("---------------------");
  
  display.print("Base:      "); display.print(angleBase); display.println(" deg");
  display.print("Shoulder:  "); display.print(angleShl);  display.println(" deg");
  display.print("Gripper:   "); display.print(angleClaw); display.println(" deg");
  
  display.display();
  
  delay(50);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, three **Potentiometers**, three **Servos**, and **SSD1306 OLED** onto the canvas.
2. Wire Potentiometers to **GPIO34, 35, 32**, Servos to **GPIO13, 12, 14**, and OLED to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the three potentiometer widgets and watch the three servos move. Verify the gripper servo does not sweep past 90 degrees.

## Expected Output
Serial Monitor:
* Values change smoothly as the dials are adjusted.

OLED Display HUD:
```
ROBOTIC ARM JOINT HUD
---------------------
Base:      90 deg
Shoulder:  90 deg
Gripper:   10 deg
```

## Expected Canvas Behavior
* Sliding Pot 1 rotates Base servo.
* Sliding Pot 2 rotates Shoulder servo.
* Sliding Pot 3 rotates Gripper servo.
* The Gripper servo widget stays within the 10°–90° range even if the potentiometer is turned completely to the right.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `CLAW_MIN_ANGLE` | Sets the minimum mechanical limit for the claw to prevent motor strain. |
| `map(valClaw, ..., MIN, MAX)` | Restricts the claw servo travel range. |
| `display.display()` | Pushes the frame buffer updates to the OLED screen. |

## Hardware & Safety Concept: Gripper Torque Overload Prevention
Servos contain a small DC motor and a feedback potentiometer. When commanded to an angle it cannot mechanically reach (e.g. if the claw jaws clamp together at 30° but the code commands 0°), the motor will stall. A stalled servo draws maximum current (up to 1.2A), generating excessive heat that will quickly burn out its motor driver circuit. Restricting servo travel in software prevents these stall conditions.

## Try This! (Challenges)
1. **Interactive Reset button**: Add a button on GPIO 4 that moves all three joints to their safe home positions (90°, 90°, 10°).
2. **Reverse Gripper Logic**: Flip the mapping of Pot Z so turning it clockwise closes the claw.
3. **Claw Jam Buzzer Alarm**: Monitor claw current (or simulate it: if claw angle is close to closed limits, play a warning chirp on a buzzer).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Third servo (Claw) does not move | Pin mapping conflict | Ensure the claw servo signal is wired to GPIO 14 and Pot Z is on GPIO 32 |
| OLED screen is blank | Initialization failure | Confirm I2C wiring SDA/SCL lines are not swapped |
| Gripper servo runs hot | Software limit set too wide | Increase `CLAW_MIN_ANGLE` to 20° or 30° so the jaws do not squeeze together too tightly |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [133 - ESP32 2-Axis Robotic Arm Joint Controller](133-esp32-2-axis-robotic-arm-joint-controller.md)
- [135 - ESP32 Robotic Arm Position Memory Log](135-esp32-robotic-arm-position-memory-log.md)
- [60 - ESP32 OLED SSD1306 Display Setup](../intermediate/60-esp32-oled-ssd1306-display-setup.md)
