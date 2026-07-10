# 128 - ESP32 Bluetooth Robot with Crash Override

Build a Bluetooth-controlled mobile robot that includes an automatic safety override, using a front-mounted HC-SR04 ultrasonic sensor to stop forward motion if a collision is imminent.

## Goal
Learn how to implement prioritized safety overrides in robot navigation controllers, interrupting user remote controls to prevent physical crashes.

## What You Will Build
An L298N driver controls two wheels (Left: 18/19/5, Right: 21/22/23). An HC-05 Bluetooth module on UART2 handles user steering inputs ('F', 'B', 'L', 'R', 'S'). A front-mounted HC-SR04 sensor measures distance. If the robot is driving forward and an obstacle enters the 20 cm collision zone, the robot stops immediately, overrides further forward commands, and sends a warning to the user.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motors (2) | `dc_motor` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth_hc05` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic_sensor` | Yes | Yes |
| External Power Supply (6–12 V) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Module | IN1 / IN2 / ENA | GPIO18 / GPIO19 / GPIO5 | Yellow/Green/Orange | Left Motor |
| L298N Module | IN3 / IN4 / ENB | GPIO21 / GPIO22 / GPIO23 | Blue/Purple/White | Right Motor |
| HC-05 Module | VCC / GND | 5V / GND | Red / Black | Power rails |
| HC-05 Module | TXD / RXD | GPIO16 / GPIO17 | Green / Yellow | Bluetooth UART |
| HC-SR04 Sensor | Trig | GPIO12 | Orange | Sonar trigger |
| HC-SR04 Sensor | Echo | GPIO13 | Yellow | Sonar echo |
| HC-SR04 Sensor | VCC / GND | 5V / GND | Red / Black | Power rails |

> **Wiring tip:** Share the 5V rail (Vin pin) for the L298N module, the HC-05, and the HC-SR04. Wire the sonar Trig/Echo pins to GPIO 12 and 13.

## Code
```cpp
// Bluetooth Robot with Crash Override
#define RX2_PIN 16
#define TX2_PIN 17

const int IN1 = 18;
const int IN2 = 19;
const int ENA = 5;

const int IN3 = 21;
const int IN4 = 22;
const int ENB = 23;

const int TRIG_PIN = 12;
const int ECHO_PIN = 13;

// Collision cutoff distance in cm
const int COLLISION_ZONE_CM = 20; 

bool isForward = false;
String inputCmd = "";

void robotForward() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  digitalWrite(ENA, HIGH);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  digitalWrite(ENB, HIGH);
  isForward = true;
}

void robotReverse() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, HIGH); digitalWrite(ENA, HIGH);
  digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH); digitalWrite(ENB, HIGH);
  isForward = false;
}

void robotPivotLeft() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, HIGH); digitalWrite(ENA, HIGH);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  digitalWrite(ENB, HIGH);
  isForward = false;
}

void robotPivotRight() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  digitalWrite(ENA, HIGH);
  digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH); digitalWrite(ENB, HIGH);
  isForward = false;
}

void robotStop() {
  digitalWrite(ENA, LOW);
  digitalWrite(ENB, LOW);
  isForward = false;
}

float getDistance() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  long duration = pulseIn(ECHO_PIN, HIGH, 20000); // 20ms timeout
  if (duration == 0) return 999.0;
  return (duration * 0.0343) / 2.0;
}

void setup() {
  Serial.begin(115200);
  Serial2.begin(9600, SERIAL_8N1, RX2_PIN, TX2_PIN);
  
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT); pinMode(ENA, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT); pinMode(ENB, OUTPUT);
  
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  robotStop();
  Serial.println("Robot Online. Telemetry & Safety Active.");
}

void loop() {
  float distance = getDistance();
  
  // 1. Crash Override Check
  // If moving forward and obstacle is too close, trigger safety halt
  if (isForward && distance <= COLLISION_ZONE_CM) {
    robotStop();
    Serial.println("!! CRASH AVOIDED: Forward motion overridden !!");
    Serial2.println("WARN: Obstacle ahead! Forward blocked.");
  }
  
  // 2. Poll Bluetooth Commands
  if (Serial2.available() > 0) {
    char cmd = (char)Serial2.read();
    
    // Ignore forward command if path is blocked
    if ((cmd == 'F' || cmd == 'f') && distance <= COLLISION_ZONE_CM) {
      Serial2.println("BLOCKED: Clear path first!");
    } else {
      switch (cmd) {
        case 'F':
        case 'f':
          robotForward();
          Serial2.println("STATE: FORWARD");
          break;
        case 'B':
        case 'b':
          robotReverse();
          Serial2.println("STATE: REVERSE");
          break;
        case 'L':
        case 'l':
          robotPivotLeft();
          Serial2.println("STATE: LEFT");
          break;
        case 'R':
        case 'r':
          robotPivotRight();
          Serial2.println("STATE: RIGHT");
          break;
        case 'S':
        case 's':
          robotStop();
          Serial2.println("STATE: STOPPED");
          break;
      }
    }
  }
  
  delay(30); // 30Hz evaluation loop
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N**, two **DC Motors**, **HC-05**, and **HC-SR04** onto the canvas.
2. Wire sonar to **GPIO12/GPIO13**, Bluetooth to **GPIO16/GPIO17**, and motors to **18, 19, 5, 21, 22, 23**.
3. Paste the code and click **Run**.
4. Set the HC-SR04 slider to 80 cm. Send `F` to drive forward.
5. Drag the distance slider below 20 cm. Watch the motors stop automatically and reject further forward commands.

## Expected Output
Serial Monitor:
```
Robot Online. Telemetry & Safety Active.
!! CRASH AVOIDED: Forward motion overridden !!
```

Bluetooth Terminal:
```
STATE: FORWARD
WARN: Obstacle ahead! Forward blocked.
BLOCKED: Clear path first!
STATE: REVERSE
```

## Expected Canvas Behavior
* Sending `F` spins both motors clockwise.
* Toggling the sonar slider below 20 cm turns both motors off immediately.
* Sending `F` while the slider is low outputs a blocked message and the wheels remain stationary.
* Sending `B` spins the wheels counter-clockwise, escaping the obstacle.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `isForward` | Helper boolean flag that tracks if the robot is currently in a forward-drive state. |
| `isForward && distance <= COLLISION_ZONE_CM` | Triggers the safety override if an obstacle enters the collision envelope. |
| `cmd == 'F' && distance <= COLLISION_ZONE_CM` | Ignores remote commands requesting forward motion while blocked. |

## Hardware & Safety Concept: Sensor Override Priorities
Safety controllers follow strict priority hierarchies. User commands are lowest priority; safety interlocks are highest. If a sensor reports a critical event (thermal runaway, limit switch hit, or sonar collision), the firmware bypasses standard execution loops to shut down actuators immediately.

## Try This! (Challenges)
1. **Dynamic Alert Siren**: Add a buzzer on GPIO 15 that emits a warning siren when the override triggers.
2. **Reverse Override Check**: Add a second rear-facing sonar (GPIO 4/15) to prevent backing up into obstacles.
3. **Emergency Brake Indicator**: Flash status LEDs (Project 118) to indicate the safety shutdown.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot runs into obstacles | Sensor field of view misaligned or slow polling | Verify sonar Trig/Echo pins; ensure loop delay is kept low (30ms) |
| Reverse doesn't function | Flag state corruption | Confirm `isForward` is set to false in the `robotReverse` function |
| Bluetooth console flooded | Serial print loop runs continuously | Ensure serial reports are sent only on changes/events, not every loop cycle |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [127 - ESP32 Bluetooth Remote Control Robot](127-esp32-bluetooth-remote-control-robot.md)
- [124 - ESP32 Obstacle Avoidance Robot](124-esp32-obstacle-avoidance-robot.md)
- [76 - ESP32 HC-SR04 Proximity Alarm](../intermediate/76-esp32-hcsr04-proximity-alarm.md)
