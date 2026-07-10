# 129 - ESP32 Robot Wall Follower

Build a mobile robot that tracks a wall parallel to its path using a side-mounted HC-SR04 ultrasonic sensor, adjusting wheel speeds proportionally to maintain a target distance.

## Goal
Learn how to implement a Proportional (P) controller feedback loop, translate error offsets to differential motor speeds, and keep a robot aligned with physical boundaries.

## What You Will Build
An L298N driver controls two wheels (Left: 18/19/5, Right: 21/22/23). An HC-SR04 sensor is mounted on the right side of the chassis (Trig: 12, Echo: 13). The robot drives forward while striving to maintain a target distance of 15.0 cm from the right-hand wall. If it drifts too far, it steers right; if it gets too close, it steers left.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motors (2) | `dc_motor` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic_sensor` | Yes | Yes |
| External Power Supply (6–12 V) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Module | IN1 / IN2 | GPIO18 / GPIO19 | Yellow / Green | Left direction bits |
| L298N Module | ENA | GPIO5 | Orange | Left speed PWM |
| L298N Module | IN3 / IN4 | GPIO21 / GPIO22 | Blue / Purple | Right direction bits |
| L298N Module | ENB | GPIO23 | White | Right speed PWM |
| L298N Module | GND | GND | Black | Shared logic ground |
| HC-SR04 Sensor | Trig | GPIO12 | Orange | Sonar trigger |
| HC-SR04 Sensor | Echo | GPIO13 | Yellow | Sonar echo input |
| HC-SR04 Sensor | VCC / GND | 5V / GND | Red / Black | Power rails |

> **Wiring tip:** The HC-SR04 is mounted pointing sideways (90 degrees to the right). Set up the ENA and ENB speed lines to GPIO 5 and 23.

## Code
```cpp
// Robot Wall Follower (Proportional Feedback)
const int IN1 = 18;
const int IN2 = 19;
const int ENA_PIN = 5;

const int IN3 = 21;
const int IN4 = 22;
const int ENB_PIN = 23;

const int TRIG_PIN = 12;
const int ECHO_PIN = 13;

// PWM Configuration
const int PWM_FREQ = 1000;
const int PWM_RES = 8;
const int LEDC_CHAN_LEFT = 0;
const int LEDC_CHAN_RIGHT = 1;

// P-Controller Parameters
const float TARGET_DISTANCE_CM = 15.0; // Distance to maintain from the wall
const float KP = 15.0;                 // Proportional feedback gain constant
const int BASE_SPEED = 150;            // Straight speed (0-255)

void setWheelSpeeds(int leftSpeed, int rightSpeed) {
  leftSpeed = constrain(leftSpeed, 0, 255);
  rightSpeed = constrain(rightSpeed, 0, 255);
  
  ledcWrite(LEDC_CHAN_LEFT, leftSpeed);
  ledcWrite(LEDC_CHAN_RIGHT, rightSpeed);
}

void robotForward() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
}

void robotStop() {
  setWheelSpeeds(0, 0);
}

float getDistance() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  long duration = pulseIn(ECHO_PIN, HIGH, 30000);
  if (duration == 0) return TARGET_DISTANCE_CM; // Return target on fail to avoid sharp turns
  return (duration * 0.0343) / 2.0;
}

void setup() {
  Serial.begin(115200);
  
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT);
  
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  // Setup LEDC PWM Channels
  ledcSetup(LEDC_CHAN_LEFT, PWM_FREQ, PWM_RES);
  ledcAttachPin(ENA_PIN, LEDC_CHAN_LEFT);
  
  ledcSetup(LEDC_CHAN_RIGHT, PWM_FREQ, PWM_RES);
  ledcAttachPin(ENB_PIN, LEDC_CHAN_RIGHT);
  
  robotStop();
  robotForward(); // Drive forward direction
  
  Serial.println("Proportional Wall Follower ready.");
}

void loop() {
  float currentDist = getDistance();
  
  // Calculate Error (Offset from target wall distance)
  float error = currentDist - TARGET_DISTANCE_CM;
  
  // Calculate steering correction
  // If too far (error > 0), steer right -> speed up left, slow down right
  // If too close (error < 0), steer left -> slow down left, speed up right
  float correction = KP * error;
  
  int leftSpeed = BASE_SPEED + (int)correction;
  int rightSpeed = BASE_SPEED - (int)correction;
  
  setWheelSpeeds(leftSpeed, rightSpeed);
  
  Serial.print("Dist: "); Serial.print(currentDist, 1);
  Serial.print(" cm | Error: "); Serial.print(error, 1);
  Serial.print(" | Left: "); Serial.print(leftSpeed);
  Serial.print(" | Right: "); Serial.println(rightSpeed);
  
  delay(60); // 16Hz control refresh rate
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N**, two **DC Motors**, and **HC-SR04** onto the canvas.
2. Wire sonar Trig/Echo to **GPIO12/GPIO13**, ENA to **GPIO5**, ENB to **GPIO23**, and inputs to **18, 19, 21, 22**.
3. Paste the code and click **Run**.
4. Adjust the sonar distance slider. At 15 cm, both motors spin at speed 150.
5. Increase distance to 25 cm. Watch the Left motor spin faster and Right motor slower.
6. Decrease distance to 5 cm. Watch the opposite correction occur.

## Expected Output
Serial Monitor:
```
Proportional Wall Follower ready.
Dist: 15.0 cm | Error: 0.0 | Left: 150 | Right: 150
Dist: 18.2 cm | Error: 3.2 | Left: 198 | Right: 102
Dist: 11.4 cm | Error: -3.6 | Left: 96 | Right: 204
```

## Expected Canvas Behavior
* While the distance slider is at 15.0, both wheels rotate at matching moderate speeds.
* Moving the slider to >15.0 makes the Left wheel spin faster and Right wheel slower (steering toward wall).
* Moving the slider to <15.0 makes the Left wheel spin slower and Right wheel faster (steering away from wall).

## Code Walkthrough
| Line | Check / Math |
| --- | --- |
| `currentDist - TARGET_DISTANCE_CM` | Evaluates error magnitude and sign. |
| `KP * error` | Scales steering correction proportionally. |
| `BASE_SPEED + correction` | Boosts or curbs wheel speeds based on calculated offsets. |

## Hardware & Safety Concept: Proportional Control Systems
A Proportional (P) controller adjusts output based on the size of the error. If the robot drifts slightly (1 cm error), the correction is small (15 PWM units). If the robot is far off course (5 cm error), the correction is large (75 PWM units). This produces smoother steering than simple on/off (bang-bang) control, which causes the robot to oscillate and swerve violently.

## Try This! (Challenges)
1. **Steering Limit bounds**: Add constraints that prevent correction values from exceeding 100 PWM units to prevent spin-outs.
2. **Dynamic Wall distance**: Connect a potentiometer (GPIO 34) to allow adjusting the target wall distance from 10 to 30 cm.
3. **Corner Turn Override**: If distance suddenly jumps to > 100 cm (indicating the wall ended or a doorway opened), execute a fixed 90° turn to follow the corner.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot steers the wrong way (towards wall when close) | Correction signs inverted | Swap the sign of `correction` in the speed calculations (e.g. `BASE_SPEED - correction` for left) |
| Robot oscillates wildly | `KP` is too high | Decrease the proportional gain constant (e.g. set `KP = 8.0`) |
| Robot reacts too slowly | `KP` is too low | Increase the gain constant or decrease the loop check delay |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [123 - ESP32 Robot Speed Control](123-esp32-robot-speed-control.md)
- [124 - ESP32 Obstacle Avoidance Robot](124-esp32-obstacle-avoidance-robot.md)
- [75 - ESP32 HC-SR04 Proximity Sensor Serial Logs](../intermediate/75-esp32-hcsr04-proximity-sensor-serial-logs.md)
