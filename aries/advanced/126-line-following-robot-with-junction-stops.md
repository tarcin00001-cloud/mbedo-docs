# 126 - Line Following Robot with Junction Stops

Implement complex track navigation on a mobile robot using three IR line tracking sensors to handle turns and execute automatic stops at junctions.

## Goal
Master multi-sensor path tracking algorithms, distinguish between curves and crossroads (T-junctions), and program automated stop actions.

## What You Will Build
A three-sensor line follower. The ARIES board monitors Left (GPIO 9), Middle (GPIO 8), and Right (GPIO 7) IR sensors. While tracking the line, the middle sensor keeps the robot aligned. If the left or right sensor clips the line, the robot steers to correct. If all three sensors detect black tape simultaneously (a crossroad or junction), the robot stops, prints "Junction Stop!" to the Serial Monitor, and remains stationary for 3 seconds before resuming travel.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| 3x IR Line Tracking Sensors (TCRT5000) | `ir_sensor` | Yes | Yes |
| L298N Motor Driver Module | `l298n` | Yes | Yes |
| 2x DC Toy Motors | `dc_motor` | Yes | Yes |
| External 6V-12V Power Source | `power_supply` | Optional | Yes (battery pack) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Left IR Sensor | VCC / GND | 3V3 / GND | Red / Black | Left sensor power |
| Left IR Sensor | DO (Digital Out) | GPIO 9 | Orange | Left tracking signal |
| Middle IR Sensor | VCC / GND | 3V3 / GND | Red / Black | Middle sensor power |
| Middle IR Sensor | DO (Digital Out) | GPIO 8 | Yellow | Center tracking signal |
| Right IR Sensor | VCC / GND | 3V3 / GND | Red / Black | Right sensor power |
| Right IR Sensor | DO (Digital Out) | GPIO 7 | Green | Right tracking signal |
| L298N Module | VCC / GND | 5V / GND | Red / Black | Power supply rails |
| L298N Module | IN1 / IN2 | GPIO 14 / 15 | Orange / Brown | Left motor control |
| L298N Module | IN3 / IN4 | GPIO 13 / 12 | Green / Blue | Right motor control |

> **Wiring tip:** Keep the three IR sensors in a neat row at the front of your chassis, spaced slightly narrower than the width of your black line tape. The center sensor should sit directly over the line, with the left and right sensors positioned just outside the tape boundaries.

## Code
```cpp
// Line Follower with Junction Stops - VEGA ARIES v3
const int LEFT_SENSOR = 9;   // Left Line Sensor on GPIO 9
const int MIDDLE_SENSOR = 8; // Middle Line Sensor on GPIO 8
const int RIGHT_SENSOR = 7;  // Right Line Sensor on GPIO 7

const int IN1 = 14;          // Left Motor Forward
const int IN2 = 15;          // Left Motor Backward
const int IN3 = 13;          // Right Motor Forward
const int IN4 = 12;          // Right Motor Backward

void setup() {
  // Configure three sensor pins as inputs
  pinMode(LEFT_SENSOR, INPUT);
  pinMode(MIDDLE_SENSOR, INPUT);
  pinMode(RIGHT_SENSOR, INPUT);

  // Configure motor control pins as outputs
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  // Initialize Serial Monitor
  Serial.begin(9600);
  Serial.println("Line Follower with Junction Stops online.");
}

void loop() {
  // Read all three line tracking sensors
  int leftState = digitalRead(LEFT_SENSOR);
  int middleState = digitalRead(MIDDLE_SENSOR);
  int rightState = digitalRead(RIGHT_SENSOR);

  // Check state combination
  if (leftState == HIGH && middleState == HIGH && rightState == HIGH) {
    // 1. Junction detected (All three sensors see black line)
    Serial.println("Junction Detected! Stopping for 3 seconds...");
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, LOW);
    delay(3000); // Stop at the junction
    
    // Drive forward slightly to clear the junction line
    Serial.println("Clearing junction...");
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, HIGH);
    digitalWrite(IN4, LOW);
    delay(400); 
  } 
  else if (leftState == LOW && middleState == HIGH && rightState == LOW) {
    // 2. Perfectly centered on line -> Go straight forward
    Serial.println("Centered: Moving Forward");
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, HIGH);
    digitalWrite(IN4, LOW);
  } 
  else if (leftState == HIGH && rightState == LOW) {
    // 3. Left sensor is over line -> Pivot Left to align
    Serial.println("Curve Left: Steering Left");
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, HIGH);
    digitalWrite(IN4, LOW);
  } 
  else if (leftState == LOW && rightState == HIGH) {
    // 4. Right sensor is over line -> Pivot Right to align
    Serial.println("Curve Right: Steering Right");
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, LOW);
  } 
  else if (leftState == LOW && middleState == LOW && rightState == LOW) {
    // 5. Out of track (All sensors see white surface) -> Stop to prevent runaway
    Serial.println("Out of Track: Stopping");
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, LOW);
  }

  delay(20); // Small interval for sensor debounce
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **L298N Motor Driver**, two **DC Toy Motors**, and three **IR Line Sensors** onto the canvas.
2. Wire the L298N module power and output terminals to the motors.
3. Wire ARIES inputs: **IN1** to **GPIO 14**, **IN2** to **GPIO 15**, **IN3** to **GPIO 13**, and **IN4** to **GPIO 12**.
4. Wire the Left Sensor DO to **GPIO 9**, Middle Sensor DO to **GPIO 8**, and Right Sensor DO to **GPIO 7**.
5. Paste the C++ code into the editor.
6. Select **Interpreted Mode** in the simulation dropdown.
7. Click **Run**. Toggle sensor states to simulate line alignment and junction stops.

## Expected Output
Serial Monitor:
```
Line Follower with Junction Stops online.
Centered: Moving Forward
Curve Left: Steering Left
Centered: Moving Forward
Junction Detected! Stopping for 3 seconds...
Clearing junction...
Centered: Moving Forward
```

## Expected Canvas Behavior
* If only the Middle Sensor widget is HIGH, both motors run forward.
* Toggling the Left Sensor HIGH (with Middle/Right LOW) stops the left motor.
* Toggling the Right Sensor HIGH (with Left/Middle LOW) stops the right motor.
* Toggling all three sensors HIGH halts both motors immediately, holds them stopped for 3 seconds, drives them forward briefly to clear the junction, and resumes monitoring.
* Toggling all sensors to LOW stops the motors.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(MIDDLE_SENSOR, INPUT)` | Configures the middle tracking sensor pin as a digital input. |
| `leftState == HIGH && middleState == HIGH && ...` | Evaluates if all three sensors are active, indicating a solid perpendicular line (T-junction). |
| `delay(3000)` | Suspends execution for 3000 ms to hold the robot completely still at the junction. |
| `delay(400)` | Drives forward for 400 ms to push the sensors past the crossroad, preventing immediate re-triggering. |

## Hardware & Safety Concept
* **Junction Detection Geometry**: In robotics, junction detection relies on sensor array spacing. When a robot crosses a T-junction or crossroad, a transverse black line crosses all three sensors. By capturing the state where all sensors read HIGH, the logic distinguishes a junction stop from standard left/right line corrections.
* **Track Recovery Safety**: If a robot moves too fast or encounters a very sharp corner, it might overshoot the line entirely. If all three sensors read LOW, the robot is lost. Stopping the motors immediately in this state prevents the robot from driving off tables or getting stuck.

## Try This! (Challenges)
1. **Junction Turn Option**: Modify the junction state so that when a junction is reached, instead of stopping, the robot turns 90 degrees to the right, then continues tracking.
2. **Speed Profile Selection**: Implement a slow tracking speed profile (e.g. PWM = 120 on enable lines) for curves, and a fast speed profile (e.g. PWM = 200) when only the center sensor is active.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot stops at curves as if they are junctions | The sensors are spaced too close together | Increase the physical distance between the Left and Right sensors so they do not both detect the same line at the same time. |
| Robot runs past junctions without stopping | Code processing is too slow or speed is too high | Decrease the base motor speed or reduce the sensor reading debounce delay (`delay(20)`). |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [125 - Line Following Robot](125-line-following-robot.md)
- [127 - Bluetooth Remote Control Robot](127-bluetooth-remote-control-robot.md)
