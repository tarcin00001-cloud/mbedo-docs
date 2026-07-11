# 124 - Obstacle Avoidance Robot

Assemble an autonomous obstacle avoidance robot using an HC-SR04 ultrasonic rangefinder, a steering servo, and DC motors.

## Goal
Implement automated navigation using distance measurements, control a servo motor to scan surroundings, and program decision-making logic for path planning.

## What You Will Build
An autonomous rover that scans its environment to avoid collisions. The robot drives forward. When the HC-SR04 ultrasonic sensor detects an obstacle closer than 20 cm, the robot stops. It then rotates a micro-servo (mounted with the ultrasonic sensor) to the right and left to scan the distances on both sides. The robot compares the readings, turns in the direction with the clearest path, centers the sensor, and continues forward.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `hc-sr04` | Yes | Yes |
| Hobby Servo Motor | `servo` | Yes | Yes |
| L298N Motor Driver Module | `l298n` | Yes | Yes |
| 2x DC Toy Motors | `dc_motor` | Yes | Yes |
| External 6V-12V Power Source | `power_supply` | Optional | Yes (battery pack) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 Sensor | VCC | 5V | Red | Sensor operating voltage (5V required) |
| HC-SR04 Sensor | Trig | GPIO 9 | Orange | Trigger pulse output from ARIES |
| HC-SR04 Sensor | Echo | GPIO 8 | Yellow | Echo signal input to ARIES |
| HC-SR04 Sensor | GND | GND | Black | Ground reference |
| Hobby Servo | Signal | GPIO 7 | Yellow | Servo PWM control signal |
| Hobby Servo | VCC | 5V | Red | Servo power supply (5V) |
| Hobby Servo | GND | GND | Black | Ground reference |
| L298N Module | VCC / GND | 5V / GND | Red / Black | Power inputs (shared GND) |
| L298N Module | IN1 / IN2 | GPIO 14 / 15 | Orange / Brown | Left motor controls |
| L298N Module | IN3 / IN4 | GPIO 13 / 12 | Green / Blue | Right motor controls |

> **Wiring tip:** The HC-SR04 and the servo motor both require 5V. On the physical board, share the 5V and GND pins using breadboard power rails. Ensure your external battery pack can handle both the L298N motor driver and the servo current draw simultaneously.

## Code
```cpp
// Obstacle Avoidance Robot - VEGA ARIES v3
#include <Servo.h>

const int TRIG_PIN = 9;  // HC-SR04 Trig on GPIO 9
const int ECHO_PIN = 8;  // HC-SR04 Echo on GPIO 8
const int SERVO_PIN = 7; // Scanning Servo on GPIO 7

const int IN1 = 14;      // Left Motor Forward
const int IN2 = 15;      // Left Motor Backward
const int IN3 = 13;      // Right Motor Forward
const int IN4 = 12;      // Right Motor Backward

Servo scanServo;

void setup() {
  // Set up ultrasonic sensor pins
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  // Set up motor driver pins
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  // Attach scanning servo
  scanServo.attach(SERVO_PIN);
  scanServo.write(90); // Center the servo (facing forward)

  // Initialize Serial Monitor
  Serial.begin(9600);
  Serial.println("Obstacle Avoidance Robot Initialized.");
  delay(1000);
}

void loop() {
  // 1. Measure forward distance
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH, 30000); // 30ms timeout (~5m max distance)
  long distance = duration * 0.034 / 2;

  // Handle timeout/invalid readings
  if (distance == 0) {
    distance = 999;
  }

  Serial.print("Forward Distance: ");
  Serial.print(distance);
  Serial.println(" cm");

  // 2. Obstacle check
  if (distance > 20) {
    // Path is clear: Drive Forward
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, HIGH);
    digitalWrite(IN4, LOW);
    delay(100); // Short delay before next measurement
  } else {
    // Obstacle detected! Stop immediately
    Serial.println("Obstacle detected! Stopping to scan...");
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, LOW);
    delay(500);

    // Scan Right (30 degrees)
    Serial.println("Scanning Right...");
    scanServo.write(30);
    delay(600); // Wait for servo to reach position
    
    digitalWrite(TRIG_PIN, LOW);
    delayMicroseconds(2);
    digitalWrite(TRIG_PIN, HIGH);
    delayMicroseconds(10);
    digitalWrite(TRIG_PIN, LOW);
    long durRight = pulseIn(ECHO_PIN, HIGH, 30000);
    long distRight = durRight * 0.034 / 2;
    if (distRight == 0) distRight = 999;

    Serial.print("Right Distance: ");
    Serial.print(distRight);
    Serial.println(" cm");

    // Scan Left (150 degrees)
    Serial.println("Scanning Left...");
    scanServo.write(150);
    delay(600); // Wait for servo to sweep
    
    digitalWrite(TRIG_PIN, LOW);
    delayMicroseconds(2);
    digitalWrite(TRIG_PIN, HIGH);
    delayMicroseconds(10);
    digitalWrite(TRIG_PIN, LOW);
    long durLeft = pulseIn(ECHO_PIN, HIGH, 30000);
    long distLeft = durLeft * 0.034 / 2;
    if (distLeft == 0) distLeft = 999;

    Serial.print("Left Distance: ");
    Serial.print(distLeft);
    Serial.println(" cm");

    // Recenter Servo
    scanServo.write(90);
    delay(400);

    // 3. Make Decision
    if (distLeft > distRight) {
      // Turn Left
      Serial.println("Turning Left...");
      digitalWrite(IN1, LOW);
      digitalWrite(IN2, HIGH);
      digitalWrite(IN3, HIGH);
      digitalWrite(IN4, LOW);
      delay(800); // Turn for 800ms
    } else {
      // Turn Right
      Serial.println("Turning Right...");
      digitalWrite(IN1, HIGH);
      digitalWrite(IN2, LOW);
      digitalWrite(IN3, LOW);
      digitalWrite(IN4, HIGH);
      delay(800); // Turn for 800ms
    }

    // Brief pause to stabilize
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, LOW);
    delay(300);
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **HC-SR04 Ultrasonic Sensor**, **Servo**, **L298N Motor Driver**, and two **DC Motors** onto the canvas.
2. Wire the L298N VCC/GND, and its outputs to the DC motors.
3. Wire the L298N inputs to ARIES: **IN1** to **GPIO 14**, **IN2** to **GPIO 15**, **IN3** to **GPIO 13**, and **IN4** to **GPIO 12**.
4. Connect the HC-SR04: **VCC** to **5V**, **GND** to **GND**, **Trig** to **GPIO 9**, and **Echo** to **GPIO 8**.
5. Connect the Servo: **VCC** to **5V**, **GND** to **GND**, and **Signal** to **GPIO 7**.
6. Paste the C++ code into the editor.
7. Select **Interpreted Mode** in the simulation dropdown.
8. Click **Run**. Use the simulated HC-SR04 slider to simulate obstacles.

## Expected Output
Serial Monitor:
```
Obstacle Avoidance Robot Initialized.
Forward Distance: 45 cm
Forward Distance: 15 cm
Obstacle detected! Stopping to scan...
Scanning Right...
Right Distance: 18 cm
Scanning Left...
Left Distance: 62 cm
Turning Left...
Forward Distance: 58 cm
```

## Expected Canvas Behavior
* While the distance slider on the HC-SR04 is above 20 cm, the motors run forward.
* When the distance slider is pulled down below 20 cm, the motors stop, the servo swings right and left, and then the motors spin in opposite directions to turn, before resuming forward travel.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `scanServo.attach(SERVO_PIN)` | Attaches the scanning servo motor control line to GPIO 7. |
| `pulseIn(ECHO_PIN, HIGH, 30000)` | Measures the duration of the incoming echo pulse in microseconds, with a 30 ms timeout. |
| `distance = duration * 0.034 / 2` | Calculates distance in cm using the speed of sound (343 m/s or 0.034 cm/µs). |
| `scanServo.write(30)` | Turns the ultrasonic sensor to point right (30 degrees). |
| `if (distLeft > distRight)` | Logically selects the direction that has more clearance. |

## Hardware & Safety Concept
* **Ultrasonic Distance Sensing**: The HC-SR04 emits an ultrasonic burst (40 kHz sound waves) when the Trig pin is driven HIGH for 10 µs. The sound wave travels through the air, hits an obstacle, and bounces back. The sensor sets the Echo pin HIGH until it receives the bounce back. By measuring the duration of this Echo pin pulse, we calculate the travel distance.
* **Power Decoupling**: Combining motors, high-current micro-servos, and sensitive sensors on a single power line can inject electrical noise and cause voltage drops. It is recommended to place a decoupling capacitor (e.g. 100 µF) close to the sensor power pins and use separate voltage regulators for power and control blocks.

## Try This! (Challenges)
1. **Back-Up Step**: Modify the logic so that if the forward obstacle is extremely close (e.g. less than 8 cm), the robot first reverses for 500 ms before scanning for a turning path.
2. **Speed Scaling Avoidance**: Adjust the speed of the motors based on distance. Go full speed at 50 cm, but gradually slow down as the distance approaches the 20 cm limit.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sensor distance readings are always 0 or out of bounds | Missing echo/trig wiring or bad sensor | Double check Trig (GPIO 9) and Echo (GPIO 8) connections. Ensure Echo is not connected to a TX pin. |
| Servo binds or shakes | Unstable current supply | Connect the servo power to a strong 5V rail and make sure the common ground is secure. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [75 - HC-SR04 Proximity Sensor Serial](../intermediate/75-hc-sr04-proximity-sensor-serial.md)
- [129 - Robot Wall Follower](129-robot-wall-follower.md)
