# 122 - ESP32 Mobile Robot Left/Right Steering

Implement differential steering algorithms to control a two-wheeled mobile robot, executing forward drives, pivot turns, and bank turns using the L298N motor driver.

## Goal
Learn how differential drive steering works, write motor control state functions, and coordinate left/right motor directions to steer a chassis.

## What You Will Build
An L298N motor driver controls two DC motors (Left and Right). GPIOs 18/19/5 control the Left Motor; GPIOs 21/22/23 control the Right Motor. The robot executes a sequence: Forward for 2 seconds, Turn Left (Pivot) for 1 second, Forward for 2 seconds, and Turn Right (Bank) for 1.5 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motors (2) | `dc_motor` | Yes | Yes |
| External Power Supply (6–12 V) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Module | IN1 / IN2 / ENA | GPIO18 / GPIO19 / GPIO5 | Yellow/Green/Orange | Left Motor control |
| L298N Module | IN3 / IN4 / ENB | GPIO21 / GPIO22 / GPIO23 | Blue/Purple/White | Right Motor control |
| L298N Module | GND | GND | Black | Ground reference |
| L298N Module | 12V (VCC) | External battery + | Red | Motor power |
| L298N Module | GND (power) | External battery − | Black | Power ground |
| Left DC Motor | Terminals | OUT1 / OUT2 | Blue / White | Left wheel |
| Right DC Motor | Terminals | OUT3 / OUT4 | Blue / White | Right wheel |

> **Wiring tip:** The wiring matches Project 121. Verify that the direction pins IN1–IN4 are mapped correctly.

## Code
```cpp
// Mobile Robot Left/Right Steering
const int IN1 = 18;
const int IN2 = 19;
const int ENA = 5;

const int IN3 = 21;
const int IN4 = 22;
const int ENB = 23;

void robotForward() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  digitalWrite(ENA, HIGH);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  digitalWrite(ENB, HIGH);
  Serial.println("Action: FORWARD");
}

void robotStop() {
  digitalWrite(ENA, LOW);
  digitalWrite(ENB, LOW);
  Serial.println("Action: STOP");
}

// 1. Pivot Turn Left: Left spins backward, Right spins forward
void robotPivotLeft() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, HIGH); digitalWrite(ENA, HIGH);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  digitalWrite(ENB, HIGH);
  Serial.println("Action: PIVOT LEFT");
}

// 2. Pivot Turn Right: Left spins forward, Right spins backward
void robotPivotRight() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  digitalWrite(ENA, HIGH);
  digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH); digitalWrite(ENB, HIGH);
  Serial.println("Action: PIVOT RIGHT");
}

// 3. Bank Turn Left (Gentle): Left is stopped, Right spins forward
void robotBankLeft() {
  digitalWrite(ENA, LOW); // Stop Left Wheel
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  digitalWrite(ENB, HIGH);
  Serial.println("Action: BANK LEFT");
}

void setup() {
  Serial.begin(115200);
  
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT); pinMode(ENA, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT); pinMode(ENB, OUTPUT);
  
  robotStop();
  Serial.println("Robot Steering ready.");
}

void loop() {
  // Drive forward
  robotForward();
  delay(2000);
  
  // Pivot Left (sharp turn)
  robotPivotLeft();
  delay(1000);
  
  // Drive forward again
  robotForward();
  delay(2000);
  
  // Bank Turn Left (gentle curve)
  robotBankLeft();
  delay(1500);
  
  // Stop and wait
  robotStop();
  delay(4000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N**, and two **DC Motors** onto the canvas.
2. Wire ENA to **GPIO5**, ENB to **GPIO23**, and inputs 1–4 to **GPIO18, 19, 21, 22**.
3. Paste the code and click **Run**.
4. Observe the motor rotations during pivot turns (spins in opposite directions) and bank turns (one motor stops).

## Expected Output
Serial Monitor:
```
Robot Steering ready.
Action: FORWARD
Action: PIVOT LEFT
Action: FORWARD
Action: BANK LEFT
Action: STOP
```

## Expected Canvas Behavior
* Forward: Both motors spin clockwise.
* Pivot Left: Left motor spins counter-clockwise, Right motor spins clockwise.
* Bank Left: Left motor stops, Right motor spins clockwise.
* Stop: Both motors stop. Cycle repeats.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `robotPivotLeft()` | Drives Left wheel backward (`IN1=LOW, IN2=HIGH`) and Right wheel forward (`IN3=HIGH, IN4=LOW`), spinning the chassis in place. |
| `robotBankLeft()` | Cuts power to the Left wheel (`ENA=LOW`) while driving the Right wheel forward, making a gentle curved turn. |
| `robotStop()` | Cuts power to both enable lines to stop all motion. |

## Hardware & Safety Concept: Pivot Turns vs Bank Turns
Differential steering offers two main turning modes:
1. **Pivot Turn (Spin Turn)**: Wheels turn in opposite directions. The center of rotation is the midpoint of the axle. This allows the robot to spin on the spot, which is ideal for tight spaces.
2. **Bank Turn (Curve Turn)**: One wheel turns faster than the other (or one wheel stops). The center of rotation is outside the chassis. This is smoother and maintains forward momentum.

## Try This! (Challenges)
1. **Pivot Right integration**: Add a `robotPivotRight()` step to the loop cycle.
2. **Backward Turn**: Write a function `robotBankLeftReverse()` that curves the robot backward to the left.
3. **Sensor Steer Trigger**: Add an IR obstacle sensor on GPIO 4. If an obstacle is detected, pivot right for 1 second, then resume forward drive.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Pivot turn causes robot to drive straight | Motor wires swapped | Verify that one motor spins backwards during pivot commands |
| Bank turn doesn't curve enough | Stopped wheel is sliding on floor | Change the stopped wheel from coast (`ENA=LOW`) to brake (`IN1=LOW, IN2=LOW, ENA=HIGH`) |
| Robot pivots when commanded forward | Left and Right motor directions mismatched | Swap the wire connections of one motor |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [121 - ESP32 Dual Motor Forward/Reverse Drive](121-esp32-dual-motor-forward-reverse-drive.md)
- [123 - ESP32 Robot Speed Control](123-esp32-robot-speed-control.md)
- [124 - ESP32 Obstacle Avoidance Robot](124-esp32-obstacle-avoidance-robot.md)
