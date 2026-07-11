# 128 - Bluetooth Robot with Crash Override

Build a remote-controlled mobile robot that integrates an HC-05 Bluetooth transceiver and an HC-SR04 ultrasonic rangefinder to automatically override user commands and prevent crashes.

## Goal
Implement priority-based sensor override logic, manage asynchronous wireless command reception, and integrate safety-critical feedback into active control loops.

## What You Will Build
A smart remote-controlled robot. The robot normally moves under manual control using commands received via Bluetooth (HC-05). However, an HC-SR04 ultrasonic sensor continuously scans the front. If the distance drops below 15 cm, the ARIES board immediately halts the robot, alerts the user with a "CRASH WARNING!" over Bluetooth, and ignores any further forward command ('F') until the path is clear or the user backs the robot away using the reverse ('B') or turn ('L'/'R') commands.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| HC-05 Bluetooth Module | `hc05` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `hc-sr04` | Yes | Yes |
| L298N Motor Driver Module | `l298n` | Yes | Yes |
| 2x DC Toy Motors | `dc_motor` | Yes | Yes |
| External 6V-12V Power Source | `power_supply` | Optional | Yes (battery pack) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-05 Module | VCC / GND | 5V / GND | Red / Black | Bluetooth module power |
| HC-05 Module | TXD | RX2 (GP21) | Blue | HC-05 TX to ARIES RX2 |
| HC-05 Module | RXD | TX2 (GP20) | Yellow | HC-05 RX to ARIES TX2 |
| HC-SR04 Sensor | VCC / GND | 5V / GND | Red / Black | Ultrasonic sensor power |
| HC-SR04 Sensor | Trig | GPIO 9 | Orange | Distance trigger pin |
| HC-SR04 Sensor | Echo | GPIO 8 | Yellow | Distance echo pin |
| L298N Module | VCC / GND | 5V / GND | Red / Black | Motor driver power |
| L298N Module | IN1 / IN2 | GPIO 14 / 15 | Orange / Brown | Left motor control |
| L298N Module | IN3 / IN4 | GPIO 13 / 12 | Green / Blue | Right motor control |

> **Wiring tip:** When combining multiple 5V devices (HC-05, HC-SR04, L298N), wire them in parallel to a shared breadboard power rail connected to the ARIES 5V and GND pins.

## Code
```cpp
// Bluetooth Robot with Crash Override - VEGA ARIES v3
const int TRIG_PIN = 9;  // HC-SR04 Trig on GPIO 9
const int ECHO_PIN = 8;  // HC-SR04 Echo on GPIO 8

const int IN1 = 14;      // Left Motor Forward
const int IN2 = 15;      // Left Motor Backward
const int IN3 = 13;      // Right Motor Forward
const int IN4 = 12;      // Right Motor Backward

char currentDirection = 'S'; // Track the robot's current moving direction

void setup() {
  // Set up ultrasonic sensor pins
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  // Set up motor driver pins
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  // Stop motors initially
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);

  // Initialize serial communication channels
  Serial.begin(9600);
  Serial2.begin(9600);

  Serial.println("Bluetooth Robot with Crash Override active.");
  Serial2.println("Control Online. Waiting for commands...");
}

void loop() {
  // 1. Measure distance to front obstacles
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH, 30000);
  long distance = duration * 0.034 / 2;

  // Handle sensor timeout
  if (distance == 0) {
    distance = 999;
  }

  // 2. Safety Check: If moving forward and obstacle is too close, trigger emergency stop
  if (distance <= 15 && (currentDirection == 'F' || currentDirection == 'f')) {
    Serial.println("SAFETY TRIPPED: Obstacle detected within 15cm! Emergency Stop!");
    Serial2.println("CRASH WARNING! Obstacle detected. Forward disabled.");
    
    // Force motors to STOP
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, LOW);
    currentDirection = 'S'; // Reset state to Stopped
  }

  // 3. Process Bluetooth command packets
  if (Serial2.available() > 0) {
    char cmd = Serial2.read();
    Serial.print("Received Remote Command: ");
    Serial.println(cmd);

    if (cmd == 'F' || cmd == 'f') {
      // Check safety condition before driving forward
      if (distance <= 15) {
        Serial2.println("ALERT: Cannot move forward! Obstacle ahead!");
      } else {
        Serial2.println("Action: Forward");
        digitalWrite(IN1, HIGH);
        digitalWrite(IN2, LOW);
        digitalWrite(IN3, HIGH);
        digitalWrite(IN4, LOW);
        currentDirection = 'F';
      }
    } 
    else if (cmd == 'B' || cmd == 'b') {
      // Reverse is always allowed to allow escape from obstacles
      Serial2.println("Action: Backward");
      digitalWrite(IN1, LOW);
      digitalWrite(IN2, HIGH);
      digitalWrite(IN3, LOW);
      digitalWrite(IN4, HIGH);
      currentDirection = 'B';
    } 
    else if (cmd == 'L' || cmd == 'l') {
      // Spin turns are allowed to rotate away from obstacles
      Serial2.println("Action: Turn Left");
      digitalWrite(IN1, LOW);
      digitalWrite(IN2, HIGH);
      digitalWrite(IN3, HIGH);
      digitalWrite(IN4, LOW);
      currentDirection = 'L';
    } 
    else if (cmd == 'R' || cmd == 'r') {
      Serial2.println("Action: Turn Right");
      digitalWrite(IN1, HIGH);
      digitalWrite(IN2, LOW);
      digitalWrite(IN3, LOW);
      digitalWrite(IN4, HIGH);
      currentDirection = 'R';
    } 
    else if (cmd == 'S' || cmd == 's') {
      Serial2.println("Action: Stop");
      digitalWrite(IN1, LOW);
      digitalWrite(IN2, LOW);
      digitalWrite(IN3, LOW);
      digitalWrite(IN4, LOW);
      currentDirection = 'S';
    }
  }

  delay(60); // Read distance roughly 15 times a second
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **HC-05 Bluetooth Module**, **HC-SR04 Ultrasonic Sensor**, **L298N Motor Driver**, and two **DC Toy Motors** onto the canvas.
2. Wire the L298N driver VCC/GND and outputs to the DC motors.
3. Wire L298N inputs to ARIES: **IN1** to **GPIO 14**, **IN2** to **GPIO 15**, **IN3** to **GPIO 13**, and **IN4** to **GPIO 12**.
4. Wire HC-05: **VCC/GND** to **5V/GND**, **TXD** to **RX2 (GP21)**, and **RXD** to **TX2 (GP20)**.
5. Wire HC-SR04: **VCC/GND** to **5V/GND**, **Trig** to **GPIO 9**, and **Echo** to **GPIO 8**.
6. Paste the C++ code into the editor.
7. Select **Interpreted Mode** and click **Run**.
8. Set the HC-SR04 slider to 50 cm. Send `'F'` in the Bluetooth terminal.
9. Drag the HC-SR04 slider below 15 cm and watch the motors halt automatically. Try sending `'F'` again to see the safety override in action, then send `'B'` to reverse.

## Expected Output
Serial Monitor (Local Debug):
```
Bluetooth Robot with Crash Override active.
Received Remote Command: F
SAFETY TRIPPED: Obstacle detected within 15cm! Emergency Stop!
Received Remote Command: F
Received Remote Command: B
```
Bluetooth Terminal (Remote App):
```
Control Online. Waiting for commands...
Action: Forward
CRASH WARNING! Obstacle detected. Forward disabled.
ALERT: Cannot move forward! Obstacle ahead!
Action: Backward
```

## Expected Canvas Behavior
* If the HC-SR04 slider is > 15 cm, motors react to all remote inputs.
* If the slider drops <= 15 cm while the motors are running forward, the motors immediately stop.
* Sending `'F'` while the slider is <= 15 cm does not start the motors.
* Sending `'B'`, `'L'`, or `'R'` turns the motors to steer away.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `char currentDirection = 'S'` | Keeps track of the robot's current moving direction state to apply conditional safety logic. |
| `distance <= 15 && ... == 'F'` | Trigger condition: checks if the robot is moving forward and gets too close to a wall. |
| `currentDirection = 'S'` | Resets the tracking state back to stopped after executing the safety shutdown. |
| `if (distance <= 15) { ... }` | Rejects any new forward user command if a wall is in front of the robot. |

## Hardware & Safety Concept
* **Critical Overrides in Autonomous Systems**: Industrial automated guided vehicles (AGVs) use safety-rated rangefinders that bypass software layers to cut motor power directly. While this code uses software-based interrupts, it mirrors this safety-first model by putting the proximity check at the top of the main execution block, overriding any pending user inputs.
* **Proximity Sensor Beam Angle**: Ultrasonic sensors emit high-frequency waves in a cone (typically 15 to 30 degrees). Small obstacles outside this direct path, or thin objects that reflect the sound away from the receiver, might not trigger the override. Combining ultrasonic rangefinders with physical limit switches (bumpers) provides redundant safety.

## Try This! (Challenges)
1. **Dynamic Brake Deceleration**: Instead of a hard digital stop, scale down the speed of the motors as the robot gets closer to the obstacle (e.g. scale speed from full to slow when distance is between 30 cm and 15 cm, then stop).
2. **Crash Override Alarm**: Connect an active buzzer to GPIO 14. If the crash override is triggered, beep the buzzer rapidly to provide physical acoustic feedback of the safety event.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot does not stop when obstacle is placed close | Sensor reading failed (timeout) | Check the trigger and echo connections. Ensure `pulseIn` is not hanging due to incorrect pin configurations. |
| Commands are ignored even when no obstacle is present | The distance calculation is returning 0 | Verify that `distance == 0` is handled (e.g., set to 999) to prevent a sensor timeout from being misread as a zero-distance crash. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [124 - Obstacle Avoidance Robot](124-obstacle-avoidance-robot.md)
- [127 - Bluetooth Remote Control Robot](127-bluetooth-remote-control-robot.md)
