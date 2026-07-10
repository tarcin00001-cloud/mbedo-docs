# 124 - ESP32 Obstacle Avoidance Robot

Build an autonomous mobile robot that detects obstacles using an HC-SR04 ultrasonic distance sensor, stops, sweeps a servo radar to scan left and right paths, and steers toward the open direction.

## Goal
Learn how to coordinate sonar distance sensors, servo positioning, and H-bridge motor drivers to implement simple autonomous navigation algorithms.

## What You Will Build
An L298N driver controls two wheels (Left: 18/19/5, Right: 21/22/23). An HC-SR04 sensor is mounted on a servo (GPIO 14) to create a scanning sonar radar. When an obstacle is detected within 25 cm, the robot stops, sweeps the sensor left and right, and turns toward the side with more open space.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motors (2) | `dc_motor` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic_sensor` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |
| External Power Supply (6–12 V) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Module | IN1 / IN2 / ENA | GPIO18 / GPIO19 / GPIO5 | Yellow/Green/Orange | Left Motor |
| L298N Module | IN3 / IN4 / ENB | GPIO21 / GPIO22 / GPIO23 | Blue/Purple/White | Right Motor |
| HC-SR04 Sensor | Trig | GPIO12 | Orange | Sonar trigger |
| HC-SR04 Sensor | Echo | GPIO13 | Yellow | Sonar echo input |
| HC-SR04 Sensor | VCC / GND | 5V / GND | Red / Black | Power rails |
| Servo Motor | Signal | GPIO14 | Brown | Radar sweep control |
| Servo Motor | VCC / GND | 5V / GND | Red / Black | Power rails |

> **Wiring tip:** Connect the sonar trigger to GPIO 12 and echo to GPIO 13. Wire the servo signal to GPIO 14. Ensure both the servo and HC-SR04 are powered from 5V (Vin).

## Code
```cpp
// Obstacle Avoidance Robot
#include <ESP32Servo.h>

const int IN1 = 18;
const int IN2 = 19;
const int ENA = 5;

const int IN3 = 21;
const int IN4 = 22;
const int ENB = 23;

const int TRIG_PIN = 12;
const int ECHO_PIN = 13;
const int SERVO_PIN = 14;

Servo radarServo;

// Collision limit in cm
const int OBSTACLE_LIMIT_CM = 25; 

void robotForward() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  digitalWrite(ENA, HIGH);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  digitalWrite(ENB, HIGH);
}

void robotStop() {
  digitalWrite(ENA, LOW);
  digitalWrite(ENB, LOW);
}

void robotPivotLeft() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, HIGH); digitalWrite(ENA, HIGH);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  digitalWrite(ENB, HIGH);
}

void robotPivotRight() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  digitalWrite(ENA, HIGH);
  digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH); digitalWrite(ENB, HIGH);
}

void robotReverse() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, HIGH); digitalWrite(ENA, HIGH);
  digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH); digitalWrite(ENB, HIGH);
}

// Read distance in centimeters
float getDistance() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  long duration = pulseIn(ECHO_PIN, HIGH, 30000); // 30ms timeout (approx 5m max)
  if (duration == 0) return 999.0;                // Return large value if timeout
  return (duration * 0.0343) / 2.0;
}

// Scans and returns distance at a specific servo angle
float scanAngle(int angle) {
  radarServo.write(angle);
  delay(500); // Wait for servo to reach position
  float dist = getDistance();
  Serial.print("Scan angle "); Serial.print(angle);
  Serial.print(": "); Serial.print(dist); Serial.println(" cm");
  return dist;
}

void setup() {
  Serial.begin(115200);
  
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT); pinMode(ENA, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT); pinMode(ENB, OUTPUT);
  
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  radarServo.attach(SERVO_PIN);
  radarServo.write(90); // Look straight ahead
  
  robotStop();
  
  Serial.println("Obstacle Avoidance Robot online.");
  delay(1000);
}

void loop() {
  float frontDist = getDistance();
  Serial.print("Front Distance: "); Serial.println(frontDist);
  
  if (frontDist <= OBSTACLE_LIMIT_CM) {
    Serial.println("!! OBSTACLE DETECTED !!");
    robotStop();
    
    // Scan left and right paths
    float distRight = scanAngle(45);  // Look right
    float distLeft  = scanAngle(135); // Look left
    
    // Return radar to center
    radarServo.write(90);
    delay(200);
    
    // Decision Tree
    if (distLeft > distRight && distLeft > OBSTACLE_LIMIT_CM) {
      Serial.println("Action: Steer Left");
      robotPivotLeft();
      delay(800); // Pivot turn duration
    } 
    else if (distRight >= distLeft && distRight > OBSTACLE_LIMIT_CM) {
      Serial.println("Action: Steer Right");
      robotPivotRight();
      delay(800);
    } 
    else {
      // Both sides blocked — back up
      Serial.println("Action: Backing Up");
      robotReverse();
      delay(1000);
      robotPivotRight(); // Turn away
      delay(800);
    }
    robotStop();
    delay(200);
  } else {
    // Path clear — drive forward
    robotForward();
    delay(50); // Small movement interval
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N**, two **DC Motors**, **HC-SR04**, and **Servo** onto the canvas.
2. Wire the inputs as listed. Wire the Servo to **GPIO14**, Trig/Echo to **GPIO12/GPIO13**.
3. Paste the code and click **Run**.
4. Slide the HC-SR04 distance widget close (e.g. 15 cm). Watch the robot stop, the servo rotate left and right to scan, and the wheels execute pivot steering.

## Expected Output
Serial Monitor:
```
Obstacle Avoidance Robot online.
Front Distance: 85.3
Front Distance: 15.2
!! OBSTACLE DETECTED !!
Scan angle 45: 80.2 cm   ← Right side clear
Scan angle 135: 12.4 cm  ← Left side blocked
Action: Steer Right
```

## Expected Canvas Behavior
* While the distance slider is > 25 cm, both motors spin forward (clockwise).
* When the distance slider drops below 25 cm, both motors stop.
* The servo widget rotates to 45° then to 135° to scan.
* Depending on which scan distance is larger, the motors execute pivot rotations (one clockwise, one counter-clockwise) to turn.
* The robot then resumes forward drive.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `pulseIn(ECHO_PIN, HIGH, 30000)` | Measures echo pulses with a 30 ms timeout to prevent locking if no return echo occurs. |
| `scanAngle(angle)` | Moves the servo radar, waits for transit, and takes a distance sample. |
| `distLeft > distRight` | Standard decision rule: steers toward the direction with more clear space. |

## Hardware & Safety Concept: Sensor Latency and Braking Distance
Robot chassis have momentum. When a collision limit is detected, the robot cannot stop instantly. A safety margin (e.g. 25 cm) accounts for:
1. **Sensor query time**: Taking a sonar sample takes ~15–30 ms.
2. **Transit deceleration**: The motor driver needs time to slow down the wheels.
3. **Decisive sweeping**: Scanning takes ~1 second during which the robot must remain stationary.

## Try This! (Challenges)
1. **Dynamic Braking**: Modify `robotStop()` to use active H-bridge braking to stop the robot faster.
2. **Backing Up alarm**: Sound a buzzer (GPIO 15) when the robot is forced to reverse.
3. **Proportional steering speed**: Adjust the turn duration based on how narrow the target pathway is.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot runs into walls before scanning | Threshold is too close | Increase `OBSTACLE_LIMIT_CM` to 30 or 35 cm |
| Servo sweeps but distance readings are identical | Sonar taking samples too fast | Add a small delay between moving the servo and calling `getDistance()` to let vibrations settle |
| Robot wiggles back and forth constantly | Sensor field of view narrow | Shield sonar sides to block reflections |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [122 - ESP32 Mobile Robot Left/Right Steering](122-esp32-mobile-robot-left-right-steering.md)
- [75 - ESP32 HC-SR04 Proximity Sensor Serial Logs](../intermediate/75-esp32-hcsr04-proximity-sensor-serial-logs.md)
- [76 - ESP32 HC-SR04 Proximity Alarm](../intermediate/76-esp32-hcsr04-proximity-alarm.md)
