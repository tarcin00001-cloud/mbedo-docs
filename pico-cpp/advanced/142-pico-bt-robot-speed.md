# 142 - Pico BT Robot Speed

Build a wireless Bluetooth robot chassis that supports variable speed mapping commands and status LCD updates.

## Goal
Learn how to parse multi-variable serial commands (e.g. speed levels) over UART and map them to PWM duty cycles on H-bridges.

## What You Will Build
A variable speed mobile robot platform:
- **HC-05 Bluetooth Module (GP0, GP1)**: Receives direction and speed commands.
- **L298N & Dual DC Motors (GP10-15)**: Drives wheels with adjustable speed.
- **16x2 I2C LCD (GP4, GP5)**: Displays current direction and speed levels.
- **Speed Commands**:
  - `'F'` (Forward), `'B'` (Backward), `'S'` (Stop)
  - `'1'` (Low speed: 40% PWM), `'2'` (Medium speed: 70% PWM), `'3'` (High speed: 100% PWM)

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| DC Motors | `dc_motor` | Yes (two motors) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 lines |
| L298N Control | ENA / ENB | GP10 / GP11 | Left/Right Speed PWM |
| L298N Control | IN1 / IN2 | GP12 / GP13 | Left wheel direction |
| L298N Control | IN3 / IN4 | GP14 / GP15 | Right wheel direction |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int ENA = 10; const int ENB = 11;
const int IN1 = 12; const int IN2 = 13;
const int IN3 = 14; const int IN4 = 15;

LiquidCrystal_I2C lcd(0x27, 16, 2);

int currentSpeed = 150; // Start at medium speed (out of 255)
char dirState[10] = "STOPPED";

void setup() {
  Serial1.begin(9600);
  
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(ENA, OUTPUT); pinMode(ENB, OUTPUT);
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT);

  // Apply default speed
  analogWrite(ENA, currentSpeed);
  analogWrite(ENB, currentSpeed);

  lcd.init();
  lcd.backlight();
  updateLCD();
}

void loop() {
  if (Serial1.available()) {
    char cmd = Serial1.read();

    // Parse speed commands
    if (cmd == '1') {
      currentSpeed = 100; // Low (40%)
      analogWrite(ENA, currentSpeed);
      analogWrite(ENB, currentSpeed);
      Serial1.println("SPEED: LOW");
    } 
    else if (cmd == '2') {
      currentSpeed = 175; // Med (70%)
      analogWrite(ENA, currentSpeed);
      analogWrite(ENB, currentSpeed);
      Serial1.println("SPEED: MED");
    } 
    else if (cmd == '3') {
      currentSpeed = 255; // High (100%)
      analogWrite(ENA, currentSpeed);
      analogWrite(ENB, currentSpeed);
      Serial1.println("SPEED: HIGH");
    }
    // Parse direction commands
    else if (cmd == 'F') {
      moveForward();
      dirState[0] = 'F'; dirState[1] = 'W'; dirState[2] = 'D'; dirState[3] = '\0';
      Serial1.println("DIR: FORWARD");
    } 
    else if (cmd == 'B') {
      moveBackward();
      dirState[0] = 'R'; dirState[1] = 'E'; dirState[2] = 'V'; dirState[3] = '\0';
      Serial1.println("DIR: REVERSE");
    } 
    else if (cmd == 'S') {
      stopRobot();
      dirState[0] = 'S'; dirState[1] = 'T'; dirState[2] = 'O'; dirState[3] = 'P'; dirState[4] = '\0';
      Serial1.println("DIR: STOPPED");
    }

    updateLCD();
  }
}

void updateLCD() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Robot: ");
  lcd.print(dirState);
  
  lcd.setCursor(0, 1);
  lcd.print("PWM Speed: ");
  lcd.print(currentSpeed * 100 / 255);
  lcd.print("%");
}

void moveForward() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
}

void moveBackward() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH);
}

void stopRobot() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);  digitalWrite(IN4, LOW);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-05**, **L298N**, **two DC Motors**, and **I2C LCD** onto the canvas.
2. Connect HC-05 to **GP0/GP1**, L298N control to **GP10-15**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Type `'F'` (Forward) in the BT terminal to spin the wheels, then type `'1'`, `'2'`, `'3'` to adjust wheel speed.

## Expected Output

Terminal (Bluetooth):
```
SPEED: MED
DIR: FORWARD
SPEED: HIGH
```

## Expected Canvas Behavior
* Press `'F'` and `'3'`: LCD reads `Robot: FWD` / `PWM Speed: 100%`. Wheels spin at full speed.
* Press `'1'`: LCD reads `Robot: FWD` / `PWM Speed: 39%`. Wheels slow down significantly.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `analogWrite(ENA, currentSpeed)` | Scales the PWM duty cycle on the H-Bridge enable pin to adjust wheel speed. |

## Hardware & Safety Concept: PWM Motor Control
H-Bridge drivers control motor speed using Pulse Width Modulation (PWM). By switching the power pin ON and OFF thousands of times per second, the average voltage delivered to the motor can be adjusted (e.g. 50% duty cycle delivers an average of 2.5V from a 5V supply), allowing smooth speed control.

## Try This! (Challenges)
1. **Directional Pivot steering**: Add commands `'L'` (Left) and `'R'` (Right) to pivot the robot chassis.
2. **Reverse Alert Beacon**: Flash a warning LED on GP16 when the robot moves backward.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motors hum but do not spin | PWM value set too low | Motors require a minimum startup voltage (typically >30% duty cycle) to overcome friction. Increase the low-speed PWM value. |

## Mode Notes
This multi-device I2C and UART project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [54 - Pico Motor Speed](../intermediate/54-pico-motor-speed.md)
- [112 - Pico Bluetooth Robot](../intermediate/112-pico-bt-robot.md)
