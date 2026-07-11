# 129 - Robot Wall Follower

Program a mobile robot to autonomously follow a wall at a constant distance using a side-mounted HC-SR04 ultrasonic sensor.

## Goal
Learn how to implement simple closed-loop distance regulation, code three-state steering corrections, and configure safety bounds to handle lost-wall conditions.

## What You Will Build
An autonomous wall-following robot. The ultrasonic rangefinder is mounted on the left side of the robot, facing outward. As the robot drives, the ARIES board reads the distance to the wall. It dynamically steers the motors: if too close (< 12 cm), it turns right; if too far (> 18 cm), it turns left; otherwise, it drives straight ahead, maintaining a target corridor of 15 cm.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `hc-sr04` | Yes | Yes |
| L298N Motor Driver Module | `l298n` | Yes | Yes |
| 2x DC Toy Motors | `dc_motor` | Yes | Yes |
| External 6V-12V Power Source | `power_supply` | Optional | Yes (battery pack) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 Sensor | VCC | 5V | Red | Proximity sensor operating power |
| HC-SR04 Sensor | Trig | GPIO 9 | Orange | Distance trigger control pin |
| HC-SR04 Sensor | Echo | GPIO 8 | Yellow | Distance echo feedback pin |
| HC-SR04 Sensor | GND | GND | Black | Shared ground connection |
| L298N Module | VCC / GND | 5V / GND | Red / Black | Motor driver power |
| L298N Module | IN1 / IN2 | GPIO 14 / 15 | Orange / Brown | Left motor control |
| L298N Module | IN3 / IN4 | GPIO 13 / 12 | Green / Blue | Right motor control |

> **Wiring tip:** Mount the HC-SR04 sensor firmly on the left side of the robot chassis, perpendicular to the forward direction of travel. Ensure the sensor sits level and is not pointing slightly upward or downward.

## Code
```cpp
// Robot Wall Follower - VEGA ARIES v3
const int TRIG_PIN = 9;  // HC-SR04 Trig on GPIO 9
const int ECHO_PIN = 8;  // HC-SR04 Echo on GPIO 8

const int IN1 = 14;      // Left Motor Forward
const int IN2 = 15;      // Left Motor Backward
const int IN3 = 13;      // Right Motor Forward
const int IN4 = 12;      // Right Motor Backward

// Wall following bounds (Target distance is 15 cm)
const int CLOSE_THRESHOLD = 12; // Turn away if closer than 12 cm
const int FAR_THRESHOLD = 18;   // Turn towards if further than 18 cm
const int MAX_WALL_DIST = 45;   // Distance indicating no wall is present

void setup() {
  // Set up ultrasonic sensor pins
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  // Set up motor driver control pins
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  // Initialize Serial Monitor
  Serial.begin(9600);
  Serial.println("Wall Follower Robot online.");
}

void loop() {
  // 1. Measure distance to the side wall
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH, 30000);
  long distance = duration * 0.034 / 2;

  // Handle sensor timeouts or out-of-range reads
  if (distance == 0) {
    distance = 999;
  }

  Serial.print("Wall Distance: ");
  Serial.print(distance);
  Serial.println(" cm");

  // 2. Control logic
  if (distance > MAX_WALL_DIST) {
    // No wall detected on the left side: Drive straight slowly searching for wall
    Serial.println("Action: No wall detected. Searching...");
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, HIGH);
    digitalWrite(IN4, LOW);
  } 
  else if (distance < CLOSE_THRESHOLD) {
    // Too close to the left wall: steer right (away from the wall)
    // Run left motor forward, turn right motor off
    Serial.println("Action: TOO CLOSE. Steering Right.");
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, LOW);
  } 
  else if (distance > FAR_THRESHOLD) {
    // Too far from the left wall: steer left (towards the wall)
    // Turn left motor off, run right motor forward
    Serial.println("Action: TOO FAR. Steering Left.");
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, HIGH);
    digitalWrite(IN4, LOW);
  } 
  else {
    // Inside the target corridor (12cm - 18cm): Drive straight forward
    Serial.println("Action: TARGET CORRIDOR. Driving Straight.");
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, HIGH);
    digitalWrite(IN4, LOW);
  }

  delay(100); // 10Hz control loop frequency
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **HC-SR04 Ultrasonic Sensor**, **L298N Motor Driver**, and two **DC Toy Motors** onto the canvas.
2. Wire the L298N inputs: **IN1** to **GPIO 14**, **IN2** to **GPIO 15**, **IN3** to **GPIO 13**, and **IN4** to **GPIO 12**. Output channels OUT1/OUT2 and OUT3/OUT4 go to the motors.
3. Wire the side-mounted HC-SR04: **VCC** to **5V**, **GND** to **GND**, **Trig** to **GPIO 9**, and **Echo** to **GPIO 8**.
4. Paste the C++ code into the editor.
5. Select **Interpreted Mode** and click **Run**.
6. Move the distance slider on the HC-SR04.
7. Observe how the motor states change: at 8 cm, the right motor stops (steering right). At 25 cm, the left motor stops (steering left). At 15 cm, both motors run forward.

## Expected Output
Serial Monitor:
```
Wall Follower Robot online.
Wall Distance: 15 cm
Action: TARGET CORRIDOR. Driving Straight.
Wall Distance: 10 cm
Action: TOO CLOSE. Steering Right.
Wall Distance: 22 cm
Action: TOO FAR. Steering Left.
```

## Expected Canvas Behavior
* Slider set to 15 cm: Both virtual motors rotate forward.
* Slider set to < 12 cm: The right motor stops, and the left motor continues rotating forward, steering the robot right.
* Slider set to 12-18 cm: Both motors spin forward.
* Slider set to > 18 cm: The left motor stops, and the right motor continues rotating forward, steering the robot left.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `distance < CLOSE_THRESHOLD` | Evaluates if the robot has drifted too close to the left wall boundary. |
| `digitalWrite(IN1, HIGH)` | Drives the left motor forward during a right turn correction. |
| `digitalWrite(IN3, LOW)` | Halts the right motor to drop speed on the right side and initiate steering. |
| `delay(100)` | Defines the sampling rate of the distance feedback controller (100 ms). |

## Hardware & Safety Concept
* **Hysteresis in Control Loops**: If we set a single threshold distance (e.g. exactly 15 cm) to switch motor states, the robot will constantly toggle steering left and right, causing severe jitter (chattering). By establishing a dead-band or target corridor (12 cm to 18 cm), we introduce hysteresis. The robot travels smoothly inside this corridor, correcting its trajectory only when it crosses the boundary lines.
* **Proximity Beam Reflection Angle**: Ultrasonic sensors function best when reflecting off flat, hard surfaces. If the robot approaches a wall at a very shallow angle, the sound wave may bounce off the wall and travel forward instead of returning to the sensor, creating an out-of-range reading. Ensure the sensor is securely and perpendicularly aligned.

## Try This! (Challenges)
1. **Right Wall Follower**: Modify the wiring and code logic so that the ultrasonic sensor is mounted on the RIGHT side of the robot and follows a right-side wall.
2. **Speed Scaling Wall Follower**: Instead of turning motors completely off to steer, implement PWM speed control (using ENA/ENB) to run one motor at speed 200 and the other at speed 120 during corrections, creating smoother, curved adjustments.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The robot steers the wrong way when correcting | Motor channels or left/right designations are reversed | Check the wiring of the left and right motor outputs. If they are correct, adjust the logic inside the `if-else` block in your code. |
| The robot drives into the wall repeatedly | Ultrasonic sensor is misaligned or pointing down | Make sure the sensor is facing directly to the side and does not clip the floor or wheel protrusions. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [76 - HC-SR04 Proximity Alarm](../intermediate/76-hc-sr04-proximity-alarm.md)
- [124 - Obstacle Avoidance Robot](124-obstacle-avoidance-robot.md)
