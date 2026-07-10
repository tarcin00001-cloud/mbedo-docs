# 114 - Pico Obstacle Avoider

Build an autonomous mobile robot that uses an ultrasonic rangefinder to detect and steer around obstacles.

## Goal
Learn how to parse distance pulse values from an HC-SR04 sensor and trigger collision avoidance steering maneuvers on dual DC motors.

## What You Will Build
An autonomous obstacle-avoiding vehicle:
- **HC-SR04 Ultrasonic Sensor (Trig GP16, Echo GP17)**: Mounted on the front of the robot.
- **L298N & Dual DC Motors (GP10-15)**: Drives forward until an obstacle is detected closer than 25 cm. When blocked, the robot stops, reverses, pivots right, and resumes forward movement.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| DC Motors | `dc_motor` | Yes (two motors) | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HC-SR04 | VCC | 5V | Sensor power |
| HC-SR04 | Trig | GP16 | Trigger pin |
| HC-SR04 | Echo | GP17 | Echo pin (requires 5V to 3.3V divider) |
| HC-SR04 | GND | GND | Ground reference |
| L298N Control | ENA / ENB | GP10 / GP11 | Left/Right motor speed |
| L298N Control | IN1 / IN2 | GP12 / GP13 | Left motor direction |
| L298N Control | IN3 / IN4 | GP14 / GP15 | Right motor direction |
| L298N | GND | GND | Shared ground |

## Code
```cpp
const int TRIG_PIN = 16;
const int ECHO_PIN = 17;

const int ENA = 10;
const int ENB = 11;
const int IN1 = 12;
const int IN2 = 13;
const int IN3 = 14;
const int IN4 = 15;

const int LIMIT_DISTANCE = 25; // Limit distance in cm

void setup() {
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  pinMode(ENA, OUTPUT);
  pinMode(ENB, OUTPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  // Set motor speed pins HIGH (full speed)
  digitalWrite(ENA, HIGH);
  digitalWrite(ENB, HIGH);
  
  digitalWrite(TRIG_PIN, LOW);
}

void loop() {
  // Generate trigger pulse
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH);
  int distance = duration * 0.0343 / 2;

  // Collision detection logic
  if (distance > 0 && distance < LIMIT_DISTANCE) {
    // Obstacle detected - stop and steer away
    stopRobot();
    delay(400);
    
    // Reverse for 800 ms
    moveBackward();
    delay(800);
    stopRobot();
    delay(200);

    // Pivot right for 600 ms
    turnRight();
    delay(600);
    stopRobot();
    delay(200);
  } else {
    // Clear path - drive forward
    moveForward();
  }

  delay(60); // Allow echo signals to settle
}

// Direction control helpers
void moveForward() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
}

void moveBackward() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH);
}

void turnRight() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  // Left wheel forward
  digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH); // Right wheel backward
}

void stopRobot() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);  digitalWrite(IN4, LOW);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-SR04 Sensor**, **L298N**, and **two DC Motors** onto the canvas.
2. Connect HC-SR04: **Trig** to **GP16**, **Echo** to **GP17** (with voltage divider). Connect L298N control lines to **GP10-15**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Move the HC-SR04 distance slider below 25 cm to simulate an obstacle, and watch the wheels reverse and turn.

## Expected Output

Terminal:
```
Simulation active. Obstacle avoider steer loops active.
```

## Expected Canvas Behavior
| Measured Obstacle | Left Wheel Action | Right Wheel Action | Robot State |
| --- | --- | --- | --- |
| Clear (> 25 cm) | Forward | Forward | Move Forward |
| Obstacle (< 25 cm) | Reverse (800ms) | Reverse (800ms) | Move Backward |
| (Next Step) | Forward (600ms) | Reverse (600ms) | Turn Right |
| Clear again | Forward | Forward | Resume Forward |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `distance > 0 && distance < LIMIT_DISTANCE` | Evaluates if the sensor detected an obstacle within the collision zone while ignoring invalid zero timeout readings. |

## Hardware & Safety Concept: Sensor Bounce Dampening
Ultrasonic sensors measure distances by sending sound waves and timing the reflection return. However, if a robot hits a wall at a sharp angle, the sound waves can bounce off sideways and never return, yielding false open-path readings. To prevent collisions from these blind spots, real robots use secondary bump switches, infrared sensors, or slow down their speed while steering.

## Try This! (Challenges)
1. **Audio Alarm Beacon**: Sound a buzzer on GP14 when the robot detects an obstacle and is reversing.
2. **Speed Scaling**: Use `analogWrite` speed controls to make the robot slow down as it gets closer to an obstacle.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot gets stuck reversing constantly | Echo connection loose | If the Echo pin fails to report, the code reads 0. Ensure the safety condition checks `distance > 0` before triggering the reverse steer. |

## Mode Notes
This multi-device control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [75 - Pico Ultrasonic Serial](../intermediate/75-pico-ultrasonic-serial.md)
- [112 - Pico Bluetooth Robot](112-pico-bt-robot.md)
- [113 - Pico Line Follower](113-pico-line-follower.md)
