# 198 - ESP32 Bluetooth Robot HUD Console

Build a Bluetooth-controlled mobile robot on the ESP32 that receives steering commands over a Bluetooth serial link, updates a local I2C LCD HUD console, and uses an HC-SR04 ultrasonic sensor to trigger an emergency stop if a collision is imminent.

## Goal
Learn how to parse serial command tokens, coordinate navigation inputs with distance sensors, implement safety overrides, and update LCD dashboards.

## What You Will Build
An L298N driver controls two DC motors (IN1: 18, IN2: 19, ENA: 5, IN3: 25, IN4: 26, ENB: 27). An HC-05 Bluetooth module is on UART2 (RX2: 16, TX2: 17). An HC-SR04 sensor is on GPIO 12/13, and a 16x2 LCD on I2C. The robot steering is controlled via Bluetooth commands ('F', 'B', 'L', 'R', 'S'). If the distance to an obstacle falls below 25 cm while moving forward, the emergency stop activates.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motors (2) | `dc_motor` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic_sensor` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| External Power Supply (6-9V) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Module | IN1 / IN2 / ENA | GPIO18 / GPIO19 / GPIO5 | Yellow/Green/Orange | Left motor speed controls |
| L298N Module | IN3 / IN4 / ENB | GPIO25 / GPIO26 / GPIO27 | Yellow/Green/Orange | Right motor speed controls |
| HC-05 Module | TXD / RXD | GPIO16 / GPIO17 | Green / Yellow | Bluetooth serial lines |
| HC-SR04 Sonar | Trig / Echo | GPIO12 / GPIO13 | Orange / Yellow | Collision avoidance lines |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Active Buzzer | VCC (+) | GPIO4 | Blue | Emergency warning horn |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the I2C bus. Power the DC motors from an external battery pack connected to the H-bridge power terminals to prevent voltage drops.

## Code
```cpp
// Bluetooth Robot HUD Console (HC-05 + HC-SR04 + L298N + LCD)
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// Motor Pins
const int IN1 = 18;
const int IN2 = 19;
const int ENA = 5;
const int IN3 = 25;
const int IN4 = 26;
const int ENB = 27;

// Sonar / Alert Pins
const int TRIG_PIN = 12;
const int ECHO_PIN = 13;
const int BUZZER_PIN = 4;

#define RX2_PIN 16
#define TX2_PIN 17

LiquidCrystal_I2C lcd(0x27, 16, 2);

// Active steering command
char activeCmd = 'S';
String statusMsg = "STOPPED";
float currentDistance = 100.0;

unsigned long lastSensorTime = 0;
const unsigned long SENSOR_INTERVAL_MS = 100; // Check distance every 100 ms

void setLeftMotor(int speed, bool forward) {
  digitalWrite(IN1, forward ? HIGH : LOW);
  digitalWrite(IN2, forward ? LOW : HIGH);
  ledcWrite(0, speed);
}

void setRightMotor(int speed, bool forward) {
  digitalWrite(IN3, forward ? HIGH : LOW);
  digitalWrite(IN4, forward ? LOW : HIGH);
  ledcWrite(1, speed);
}

void stopRobot() {
  ledcWrite(0, 0);
  ledcWrite(1, 0);
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
}

void setup() {
  Serial.begin(115200);
  Serial2.begin(9600, SERIAL_8N1, RX2_PIN, TX2_PIN); // Bluetooth Serial link
  
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  digitalWrite(BUZZER_PIN, LOW);
  
  // Setup PWM channels
  ledcSetup(0, 2000, 8);
  ledcAttachPin(ENA, 0);
  ledcSetup(1, 2000, 8);
  ledcAttachPin(ENB, 1);
  
  stopRobot();
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("BT Robot Console");
  lcd.setCursor(0, 1);
  lcd.print("Ready for BT...");
  
  delay(1500);
  lcd.clear();
}

void loop() {
  unsigned long now = millis();
  
  // 1. Measure Distance periodically
  if (now - lastSensorTime >= SENSOR_INTERVAL_MS) {
    digitalWrite(TRIG_PIN, LOW);
    delayMicroseconds(2);
    digitalWrite(TRIG_PIN, HIGH);
    delayMicroseconds(10);
    digitalWrite(TRIG_PIN, LOW);
    
    long duration = pulseIn(ECHO_PIN, HIGH, 20000); // 20ms timeout (~3.4m)
    currentDistance = (duration > 0) ? (duration * 0.0343 / 2.0) : 400.0;
    
    lastSensorTime = now;
  }
  
  // 2. Read Bluetooth Serial steering commands
  if (Serial2.available() > 0) {
    char cmd = Serial2.read();
    
    // Filter command characters
    if (cmd == 'F' || cmd == 'B' || cmd == 'L' || cmd == 'R' || cmd == 'S') {
      activeCmd = cmd;
    }
  }
  
  // 3. Evaluate Emergency Collision Avoidance Stop
  // If moving forward and obstacle is closer than 25 cm
  if (activeCmd == 'F' && currentDistance < 25.0) {
    Serial.println("!!! EMERGENCY STOP: OBSTACLE DETECTED !!!");
    activeCmd = 'S'; // Override command to Stop
    statusMsg = "SAFE STOP";
    
    stopRobot();
    
    // Sound warning buzzer
    digitalWrite(BUZZER_PIN, HIGH);
    delay(200);
    digitalWrite(BUZZER_PIN, LOW);
  }
  
  // 4. Actuate Motors based on command
  switch (activeCmd) {
    case 'F':
      setLeftMotor(180, true);
      setRightMotor(180, true);
      statusMsg = "FORWARD";
      break;
    case 'B':
      setLeftMotor(180, false);
      setRightMotor(180, false);
      statusMsg = "BACKWARD";
      break;
    case 'L':
      setLeftMotor(150, false);
      setRightMotor(150, true);
      statusMsg = "LEFT SPIN";
      break;
    case 'R':
      setLeftMotor(150, true);
      setRightMotor(150, false);
      statusMsg = "RIGHT SPIN";
      break;
    case 'S':
      if (statusMsg != "SAFE STOP") {
        statusMsg = "STOPPED";
      }
      stopRobot();
      break;
  }
  
  // 5. Update LCD HUD Console
  lcd.setCursor(0, 0);
  lcd.print("CMD: "); lcd.print(activeCmd);
  lcd.print(" Dist: "); lcd.print(currentDistance, 1); lcd.print("cm ");
  
  lcd.setCursor(0, 1);
  lcd.print("HUD: "); lcd.print(statusMsg);
  lcd.print("       "); // Clear trailing space
  
  delay(20);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N**, two **Motors**, **HC-SR04**, **HC-05**, **Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Wire inputs, Sonar to **GPIO12/GPIO13**, HC-05 RX/TX to **GPIO16/GPIO17**, Buzzer to **GPIO4**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Type `F` in the Bluetooth terminal. Watch the motor widgets spin forward.
5. Slide the sonar distance widget below 25 cm. Watch the motors stop instantly, the buzzer beep, and the LCD display `SAFE STOP`.

## Expected Output
Serial Monitor:
```
CMD: F Dist: 85.2cm (FORWARD)
CMD: F Dist: 52.1cm (FORWARD)
!!! EMERGENCY STOP: OBSTACLE DETECTED !!!
CMD: S Dist: 22.4cm (SAFE STOP)
```

LCD Display:
```
CMD: F Dist: 52.1cm
HUD: FORWARD
```

LCD Display (triggered stop):
```
CMD: S Dist: 22.4cm
HUD: SAFE STOP
```

## Expected Canvas Behavior
* At startup, the LCD initializes, and the motors are stopped.
* Sending `F` over the Bluetooth Serial terminal starts the motor widgets spinning clockwise.
* Lowering the sonar distance widget below 25 cm stops the motors, pulses the buzzer widget green, and updates the LCD to `HUD: SAFE STOP`.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Serial2.read()` | Reads steering commands sent from the remote Bluetooth terminal. |
| `currentDistance < 25.0` | Checks if the path is clear, triggering the stop if an obstacle is too close. |
| `activeCmd = 'S'` | Overrides the active command to stop the motors immediately. |

## Hardware & Safety Concept: Active Braking and Collision Avoidance
Industrial vehicles use active collision avoidance. While remote operators have steering controls, local safety sensors (like ultrasonic or LiDAR sensors) monitor the surroundings. If the vehicle is moving forward and a collision is imminent, the local controller overrides the operator's commands, stopping the vehicle. This is critical in automated guided vehicles (AGVs) to protect hardware.

## Try This! (Challenges)
1. **Speed modulation command**: Adjust the motor speed using numbers '1' (slow) to '9' (fast) received over Bluetooth.
2. **Reverse Safety sensor**: Add a second ultrasonic sensor (GPIO 14/15) to check for obstacles when reversing.
3. **Compass steering lock**: Combine a digital compass (Project 148) to lock the robot's heading during Bluetooth runs.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motors do not spin | ENA/ENB jumpers missing | Verify that ENA and ENB speed control lines are wired correctly to the ESP32 PWM pins |
| Bluetooth commands ignored | Pin mapping conflict | Verify that the HC-05 RX is connected to TX2 (GPIO 17) and TX to RX2 (GPIO 16) |
| Sonar distance reads static 0 | Echo pulse timed out | Verify that the echo pin is connected to GPIO 13 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [127 - ESP32 Bluetooth Remote Control Robot](../intermediate/127-esp32-bluetooth-remote-control-robot.md)
- [124 - ESP32 Obstacle Avoidance Robot](../intermediate/124-esp32-obstacle-avoidance-robot.md)
- [191 - ESP32 Autonomous Robot HUD Node](191-esp32-autonomous-robot-hud-node.md)
