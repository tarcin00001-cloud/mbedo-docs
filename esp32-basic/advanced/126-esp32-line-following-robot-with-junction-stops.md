# 126 - ESP32 Line Following Robot with Junction Stops

Enhance an autonomous line-following robot with junction detection, stopping the robot when a horizontal crossline (T-junction or stop line) is scanned, and requiring a push-button press to resume tracking.

## Goal
Learn how to implement intersection detection in line-following logic, manage state machines with button interrupts or digital polls, and execute transit overrides.

## What You Will Build
An L298N driver controls two wheels. Two front-mounted IR line sensors (Left: GPIO 12, Right: GPIO 13) trace a track. A push button on GPIO 4 acts as a restart switch. When both sensors detect black (HIGH), the robot stops (at a junction). The robot remains stationary until the user presses the button, which commands it to cross the junction and continue line following.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motors (2) | `dc_motor` | Yes | Yes |
| IR Line Tracking Sensors (2) | `line_sensor` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |
| External Power Supply (6–12 V) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Module | IN1 / IN2 / ENA | GPIO18 / GPIO19 / GPIO5 | Yellow/Green/Orange | Left Motor |
| L298N Module | IN3 / IN4 / ENB | GPIO21 / GPIO22 / GPIO23 | Blue/Purple/White | Right Motor |
| Left Line Sensor | DO | GPIO12 | Blue | Left track sensor |
| Right Line Sensor| DO | GPIO13 | Yellow | Right track sensor |
| Push Button | Pin 1 / Pin 2 | 3V3 / GPIO4 | Red / Yellow | Resume trigger |
| 10 kΩ Resistor | Leg 1 / Leg 2 | GPIO4 / GND | White / Black | Pull-down for button |
| Shared power | VCC / GND | 3V3 / GND | Red / Black | Sensor power lines |

> **Wiring tip:** Connect the resume button to GPIO 4 with a 10 kΩ pull-down resistor. The rest of the motor and line sensor connections are identical to Project 125.

## Code
```cpp
// Line Following Robot with Junction Stops
const int IN1 = 18;
const int IN2 = 19;
const int ENA = 5;

const int IN3 = 21;
const int IN4 = 22;
const int ENB = 23;

const int SENSOR_LEFT = 12;
const int SENSOR_RIGHT = 13;
const int RESUME_BTN = 4;

const int BLACK_LINE = HIGH;
const int WHITE_FLOOR = LOW;

// Robot States
enum RobotState { IDLE, TRACKING, JUNCTION_STOP };
RobotState currentState = TRACKING;

void robotForward() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  digitalWrite(ENA, HIGH);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  digitalWrite(ENB, HIGH);
}

void robotStop() {
  digitalWrite(ENA, LOW);
  digitalWrite(ENB, LOW);
}

void robotTurnLeft() {
  digitalWrite(ENA, LOW);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  digitalWrite(ENB, HIGH);
}

void robotTurnRight() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  digitalWrite(ENA, HIGH);
  digitalWrite(ENB, LOW);
}

void setup() {
  Serial.begin(115200);
  
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT); pinMode(ENA, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT); pinMode(ENB, OUTPUT);
  
  pinMode(SENSOR_LEFT, INPUT);
  pinMode(SENSOR_RIGHT, INPUT);
  pinMode(RESUME_BTN, INPUT);
  
  robotStop();
  Serial.println("Junction Line Follower ready. System Tracking.");
}

void loop() {
  int leftState = digitalRead(SENSOR_LEFT);
  int rightState = digitalRead(SENSOR_RIGHT);
  bool resumePressed = (digitalRead(RESUME_BTN) == HIGH);
  
  switch (currentState) {
    case TRACKING:
      // 1. Check for Junction Marker (Both sensors hit black line)
      if (leftState == BLACK_LINE && rightState == BLACK_LINE) {
        Serial.println(">> JUNCTION DETECTED: Stopping robot <<");
        robotStop();
        currentState = JUNCTION_STOP;
      } 
      // 2. Standard tracking logic
      else if (leftState == WHITE_FLOOR && rightState == WHITE_FLOOR) {
        robotForward();
      } 
      else if (leftState == BLACK_LINE && rightState == WHITE_FLOOR) {
        robotTurnLeft();
      } 
      else if (leftState == WHITE_FLOOR && rightState == BLACK_LINE) {
        robotTurnRight();
      }
      break;
      
    case JUNCTION_STOP:
      robotStop();
      // Wait for user resume press
      if (resumePressed) {
        Serial.println("Resume pressed. Crossing junction...");
        
        // Drive forward briefly to cross the black line junction marker
        robotForward();
        delay(400); // Wait for sensors to clear the cross line
        
        currentState = TRACKING;
      }
      break;
      
    case IDLE:
      robotStop();
      break;
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N**, two **DC Motors**, two **Line Sensors**, and **Button** onto the canvas.
2. Wire the components as listed. Connect Button to **GPIO4**.
3. Paste the code and click **Run**.
4. Set both line sensors to HIGH (simulating a black crossline). Watch the robot stop.
5. Click the button to restart the tracking loop.

## Expected Output
Serial Monitor:
```
Junction Line Follower ready. System Tracking.
>> JUNCTION DETECTED: Stopping robot <<
Resume pressed. Crossing junction...
```

## Expected Canvas Behavior
* While tracking normal lines, wheels rotate forward or steer left/right.
* Toggling both line sensors HIGH immediately stops both motor widgets.
* The motors stay off until the button widget is clicked, which causes the motors to spin forward for 400 ms and then resume tracking.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `currentState = JUNCTION_STOP` | Shifts the state machine to lock inputs when a junction is hit. |
| `delay(400)` | Keeps the robot moving forward during junction crossing to clear the sensors past the intersection line. |
| `digitalRead(RESUME_BTN)` | Polls the manual override push-button input. |

## Hardware & Safety Concept: AGV Line Navigation and Safety Stops
Automated Guided Vehicles (AGVs) in smart warehouses follow painted magnetic or optical floor tracks. Cross-lines indicate loading bays, charging areas, or intersecting lanes. The AGV stops at these points to check for other warehouse traffic before resuming transit.

## Try This! (Challenges)
1. **Status Buzzer Alert**: Add a buzzer on GPIO 15 that beeps twice when a junction is hit, and once when resuming.
2. **Junction Choice Menu**: Add a second button on GPIO 14. Button A turns left at junctions, Button B turns right.
3. **Target Trip Counter**: Automatically count and display the number of junctions crossed on an I2C LCD.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot does not stop at junctions | One of the sensors did not register the black line | Verify both line sensors are aligned horizontally and pass over the crossline together |
| Robot stops and immediately restarts | Button bouncing | Add software debouncing or increase the button lockout time |
| Crossing transit distance too short | `delay(400)` is insufficient | Increase the delay to allow the robot physical time to clear the crossline width |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [125 - ESP32 Line Following Robot](125-esp32-line-following-robot.md)
- [122 - ESP32 Mobile Robot Left/Right Steering](122-esp32-mobile-robot-left-right-steering.md)
- [63 - ESP32 TM1637 Counter Display](63-esp32-tm1637-counter-display.md)
