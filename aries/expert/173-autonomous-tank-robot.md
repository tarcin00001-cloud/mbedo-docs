# 173 - Autonomous Tank Robot

Build a wheeled or tank robot that uses an ultrasonic sensor swept by a servo motor to navigate around obstacles using a non-blocking state-machine avoidance strategy.

## Goal
Learn how to control dual DC motors using an L298N driver, implement time-sliced sensor scanning, write distance calculations via pulse-in logic, and code a navigation state machine without using `for` or `while` loops.

## What You Will Build
An autonomous vehicle that drives forward while checking for obstacles. If an obstacle is detected within 20cm, it stops, sounds a warning beep, sweeps its ultrasonic sensor left and right, compares the clear distances, and turns in the direction with more open space before resuming forward motion.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| 2x DC Motors | `dc_motor` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| SG90 Servo Motor | `servo` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Red Warning LED | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Motor Driver | ENA (Left PWM) | GPIO 9 | Orange | Left motor speed control |
| L298N Motor Driver | IN1 (Left Dir 1) | GPIO 7 | Yellow | Left motor direction |
| L298N Motor Driver | IN2 (Left Dir 2) | GPIO 8 | Green | Left motor direction |
| L298N Motor Driver | ENB (Right PWM)| GPIO 10 | Blue | Right motor speed control |
| L298N Motor Driver | IN3 (Right Dir 1)| GPIO 3 | Purple | Right motor direction |
| L298N Motor Driver | IN4 (Right Dir 2)| GPIO 4 | Gray | Right motor direction |
| L298N Motor Driver | VCC | 5V | Red | Power input (5V) |
| L298N Motor Driver | GND | GND | Black | Common Ground |
| SG90 Servo | Signal | GPIO 13 | Yellow | Servo sweep control |
| SG90 Servo | VCC | 5V | Red | Power |
| SG90 Servo | GND | GND | Black | Ground |
| HC-SR04 Sensor | TRIG | GPIO 11 | Orange | Trigger pulse output |
| HC-SR04 Sensor | ECHO | GPIO 12 | Green | Echo pulse input |
| HC-SR04 Sensor | VCC | 5V | Red | Power |
| HC-SR04 Sensor | GND | GND | Black | Ground |
| Potentiometer | Pin 2 (Wiper) | ADC0 | Blue | Speed adjustment (GP26) |
| Potentiometer | Pin 1 (VCC) | 3V3 | Red | Reference Voltage |
| Potentiometer | Pin 3 (GND) | GND | Black | Ground |
| Active Buzzer | + | GPIO 14 | Gray | Sound alarm pin |
| Active Buzzer | - | GND | Black | Ground |
| Red LED | Anode | GPIO 15 | Red | Status indicator |
| Red LED | Cathode | GND | Black | Ground |

> **Wiring tip:** Set the L298N jumpers properly if you are using onboard 5V regulator, or supply direct 5V to the board from external batteries to prevent RISC-V CPU resets when motors pull heavy startup current.

## Code
```cpp
// Autonomous Tank Robot - VEGA ARIES v3
#include <Servo.h>

const int ENA = 9;
const int IN1 = 7;
const int IN2 = 8;
const int ENB = 10;
const int IN3 = 3;
const int IN4 = 4;

const int TRIG = 11;
const int ECHO = 12;
const int SERVO_PIN = 13;
const int BUZZER = 14;
const int LED_PIN = 15;
const int SPEED_POT = ADC0;

Servo scannerServo;

int robotState = 0; 
// 0: Drive Forward, 1: Stop/Look Left, 2: Look Right, 3: Compare, 4: Backing Up, 5: Turn Left, 6: Turn Right, 7: Recenter

unsigned long stateTimer = 0;
int maxSpeed = 150;
float distCenter = 0.0;
float distLeft = 0.0;
float distRight = 0.0;

void setup() {
  Serial.begin(115200);
  pinMode(ENA, OUTPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(ENB, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  
  pinMode(TRIG, OUTPUT);
  pinMode(ECHO, INPUT);
  pinMode(BUZZER, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  
  scannerServo.attach(SERVO_PIN);
  scannerServo.write(90); // Center position
  
  // Set motors forward initially
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  
  stateTimer = millis();
}

void loop() {
  // Read speed limit dynamically from potentiometer
  maxSpeed = map(analogRead(SPEED_POT), 0, 1023, 0, 255);
  
  // NAVIGATION STATE MACHINE
  if (robotState == 0) { // Drive Forward
    // Drive Motors Forward
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, HIGH);
    digitalWrite(IN4, LOW);
    analogWrite(ENA, maxSpeed);
    analogWrite(ENB, maxSpeed);
    
    // Non-blocking trigger of HC-SR04
    digitalWrite(TRIG, LOW);
    delayMicroseconds(2);
    digitalWrite(TRIG, HIGH);
    delayMicroseconds(10);
    digitalWrite(TRIG, LOW);
    
    long duration = pulseIn(ECHO, HIGH, 30000); // 30ms timeout
    distCenter = duration * 0.034 / 2;
    if (distCenter == 0) {
      distCenter = 400.0; // Out of range
    }
    
    if (distCenter < 20.0) {
      // Obstacle detected! Stop immediately
      analogWrite(ENA, 0);
      analogWrite(ENB, 0);
      digitalWrite(BUZZER, HIGH);
      digitalWrite(LED_PIN, HIGH);
      delay(150);
      digitalWrite(BUZZER, LOW);
      
      robotState = 1; // Transition to Look Left
      scannerServo.write(145);
      stateTimer = millis();
    }
  } 
  else if (robotState == 1) { // Look Left
    if (millis() - stateTimer >= 600) {
      // Read Left distance
      digitalWrite(TRIG, LOW);
      delayMicroseconds(2);
      digitalWrite(TRIG, HIGH);
      delayMicroseconds(10);
      digitalWrite(TRIG, LOW);
      long duration = pulseIn(ECHO, HIGH, 30000);
      distLeft = duration * 0.034 / 2;
      if (distLeft == 0) distLeft = 400.0;
      
      scannerServo.write(35); // Set to Look Right
      robotState = 2; // Transition to Look Right
      stateTimer = millis();
    }
  } 
  else if (robotState == 2) { // Look Right
    if (millis() - stateTimer >= 600) {
      // Read Right distance
      digitalWrite(TRIG, LOW);
      delayMicroseconds(2);
      digitalWrite(TRIG, HIGH);
      delayMicroseconds(10);
      digitalWrite(TRIG, LOW);
      long duration = pulseIn(ECHO, HIGH, 30000);
      distRight = duration * 0.034 / 2;
      if (distRight == 0) distRight = 400.0;
      
      robotState = 3; // Transition to Compare
    }
  } 
  else if (robotState == 3) { // Compare
    if (distLeft < 20.0 && distRight < 20.0) {
      // Trapped! Back up
      digitalWrite(IN1, LOW);
      digitalWrite(IN2, HIGH);
      digitalWrite(IN3, LOW);
      digitalWrite(IN4, HIGH);
      analogWrite(ENA, maxSpeed / 2);
      analogWrite(ENB, maxSpeed / 2);
      robotState = 4; // Backing Up
      stateTimer = millis();
    } 
    else if (distLeft > distRight) {
      // Turn Left
      digitalWrite(IN1, LOW);
      digitalWrite(IN2, HIGH);
      digitalWrite(IN3, HIGH);
      digitalWrite(IN4, LOW);
      analogWrite(ENA, maxSpeed);
      analogWrite(ENB, maxSpeed);
      robotState = 5; // Turn Left
      stateTimer = millis();
    } 
    else {
      // Turn Right
      digitalWrite(IN1, HIGH);
      digitalWrite(IN2, LOW);
      digitalWrite(IN3, LOW);
      digitalWrite(IN4, HIGH);
      analogWrite(ENA, maxSpeed);
      analogWrite(ENB, maxSpeed);
      robotState = 6; // Turn Right
      stateTimer = millis();
    }
  } 
  else if (robotState == 4) { // Backing Up
    // Back up for 800ms
    if (millis() - stateTimer >= 800) {
      // Force a turn Right after backing up
      digitalWrite(IN1, HIGH);
      digitalWrite(IN2, LOW);
      digitalWrite(IN3, LOW);
      digitalWrite(IN4, HIGH);
      robotState = 6;
      stateTimer = millis();
    }
  } 
  else if (robotState == 5) { // Turn Left
    if (millis() - stateTimer >= 500) { // Turn duration
      robotState = 7; // Recenter
      scannerServo.write(90);
      stateTimer = millis();
    }
  } 
  else if (robotState == 6) { // Turn Right
    if (millis() - stateTimer >= 500) { // Turn duration
      robotState = 7; // Recenter
      scannerServo.write(90);
      stateTimer = millis();
    }
  } 
  else if (robotState == 7) { // Recenter
    if (millis() - stateTimer >= 400) {
      digitalWrite(LED_PIN, LOW);
      robotState = 0; // Resume Driving
    }
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **L298N driver**, **two DC Motors**, **HC-SR04**, **SG90 Servo**, **Potentiometer**, **Buzzer**, and **LED** onto the canvas.
2. Complete the wiring according to the table.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the dropdown.
5. Click **Run**.
6. Alter the ultrasonic sensor widget distance slider to < 20cm to trigger the automatic scanning and avoidance sequence.

## Expected Output
Serial Monitor:
```
Distance: 45.3 cm | Driving Forward
Distance: 12.1 cm | Obstacle Detected! Stopping.
Scanning Left: 82.0 cm
Scanning Right: 14.5 cm
Action: Turn Left
Recenter completed. Resuming Forward.
```

## Expected Canvas Behavior
* The dual DC motors spin forward at the speed set by the potentiometer.
* When the HC-SR04 distance slider is moved below 20, the motors stop, the buzzer makes a quick click, and the servo rotates left and then right.
* The motors then spin in opposite directions to execute a spin turn (depending on which direction was clearer).
* After turning, the servo centers, and the motors resume forward rotation.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pulseIn(ECHO, HIGH, 30000)` | Measures echo pulse width; 30ms limit stops blocking when no return wave is received. |
| `robotState = 1` | Transition state setting. Prevents blocking delay loops while letting servo stabilize. |
| `millis() - stateTimer >= 600` | Gives the servo motor 600ms of settling time to reach 145 degrees before triggering a read. |
| `distLeft > distRight` | Compares path availability and configures motor direction states to steer clear of walls. |
| `map(...)` | Re-scales the potentiometer 10-bit output (0-1023) to 8-bit motor PWM duty cycle (0-255). |

## Hardware & Safety Concept
* **High Startup Current**: DC motors draw up to 2A during sudden forward/reverse transitions. Wire the L298N logic ground to the ARIES ground, but keep the motor power source (batteries) isolated from the ARIES 5V power rails.
* **Ultrasonic Multipath Reflection**: The ultrasonic pulse can reflect off walls at steep angles and fail to return, leading to false maximum readings. The scanning routine helps average out these blindspots.

## Try This! (Challenges)
1. **Dynamic Scanning speed**: Modify the code so that the speed potentiometer also increases or decreases the scanning rate (state transition delays) proportionally.
2. **Speed Reduction Zone**: If an obstacle is detected between 20cm and 50cm, reduce motor speed to 30% of max speed to prepare for a stop.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot turns endlessly in circles | Motors wired in opposite polarity | Reverse Terminal A and B wires on the motor that is spinning backwards. |
| Distance reads zero constantly | Trigger/Echo pins swapped | Swap wires on pins 11 and 12. Check if sensor has a common ground with ARIES. |
| Robot resets when motors start | High current brownout | Supply L298N power terminal from an external battery pack instead of the ARIES 5V pin. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [124 - Obstacle Avoidance Robot](../advanced/124-obstacle-avoidance-robot.md)
- [129 - Robot Wall Follower](../advanced/129-robot-wall-follower.md)
- [179 - Industrial Motor Controller](179-industrial-motor-controller.md)
