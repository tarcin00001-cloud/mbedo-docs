# 125 - ESP32 Line Following Robot

Build an autonomous line-following robot using two infrared (IR) reflective line tracking sensors to steer a dual-motor L298N chassis along a black track.

## Goal
Learn how to read digital infrared reflective sensors, design line tracking logic tables, and implement differential steering response loops.

## What You Will Build
An L298N motor driver controls two wheels (Left: 18/19/5, Right: 21/22/23). Two IR line tracking sensors are mounted at the front (Left: GPIO 12, Right: GPIO 13). The robot steers left or right to keep the black track centered between the two sensors.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motors (2) | `dc_motor` | Yes | Yes |
| IR Line Tracking Sensors (2) | `line_sensor` | Yes | Yes |
| External Power Supply (6–12 V) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Module | IN1 / IN2 / ENA | GPIO18 / GPIO19 / GPIO5 | Yellow/Green/Orange | Left Motor |
| L298N Module | IN3 / IN4 / ENB | GPIO21 / GPIO22 / GPIO23 | Blue/Purple/White | Right Motor |
| Left Line Sensor | DO | GPIO12 | Blue | Left track sensor |
| Left Line Sensor | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| Right Line Sensor| DO | GPIO13 | Yellow | Right track sensor |
| Right Line Sensor| VCC / GND | 3V3 / GND | Red / Black | Power rails |

> **Wiring tip:** Connect the Left line sensor output to GPIO 12 and the Right line sensor to GPIO 13. The line tracking sensors are digital inputs.

## Code
```cpp
// Line Following Robot
const int IN1 = 18;
const int IN2 = 19;
const int ENA = 5;

const int IN3 = 21;
const int IN4 = 22;
const int ENB = 23;

const int SENSOR_LEFT = 12;
const int SENSOR_RIGHT = 13;

// Sensor logic definitions
// Most IR line sensors output HIGH on black lines (no reflection)
// and LOW on white floors (high reflection)
const int BLACK_LINE = HIGH;
const int WHITE_FLOOR = LOW;

void robotForward() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  digitalWrite(ENA, HIGH);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  digitalWrite(ENB, HIGH);
}

void robotStop() {
  digitalWrite(ENA, LOW);
  digitalWrite(ENB, LOW);
}

// Turn Left: Stop Left wheel, run Right wheel forward
void robotTurnLeft() {
  digitalWrite(ENA, LOW); // Coast/stop left
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  digitalWrite(ENB, HIGH);
}

// Turn Right: Run Left wheel forward, stop Right wheel
void robotTurnRight() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  digitalWrite(ENA, HIGH);
  digitalWrite(ENB, LOW); // Coast/stop right
}

void setup() {
  Serial.begin(115200);
  
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT); pinMode(ENA, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT); pinMode(ENB, OUTPUT);
  
  pinMode(SENSOR_LEFT, INPUT);
  pinMode(SENSOR_RIGHT, INPUT);
  
  robotStop();
  Serial.println("Line Following Robot ready.");
}

void loop() {
  int leftState = digitalRead(SENSOR_LEFT);
  int rightState = digitalRead(SENSOR_RIGHT);
  
  // Line Tracking Logic Table
  // Case 1: Both sensors on white floor -> Go straight
  if (leftState == WHITE_FLOOR && rightState == WHITE_FLOOR) {
    robotForward();
  }
  // Case 2: Left sensor hits black line -> Steer Left to center track
  else if (leftState == BLACK_LINE && rightState == WHITE_FLOOR) {
    robotTurnLeft();
  }
  // Case 3: Right sensor hits black line -> Steer Right to center track
  else if (leftState == WHITE_FLOOR && rightState == BLACK_LINE) {
    robotTurnRight();
  }
  // Case 4: Both sensors hit black line -> Stop (end of track or intersection)
  else if (leftState == BLACK_LINE && rightState == BLACK_LINE) {
    robotStop();
  }
  
  delay(10); // 100Hz scan loop
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N**, two **DC Motors**, and two **Line Sensors** onto the canvas.
2. Wire Left Sensor to **GPIO12**, Right Sensor to **GPIO13**, and motor controls to **18, 19, 5, 21, 22, 23**.
3. Paste the code and click **Run**.
4. Click the line sensor widgets to toggle states (black/white) and observe the motor steering responses.

## Expected Output
Serial Monitor:
```
Line Following Robot ready.
(Robot drives straight when both read WHITE)
(Left motor stops, Right motor spins when Left reads BLACK)
```

## Expected Canvas Behavior
* Both sensors LOW (white): Both motors spin clockwise (forward).
* Left sensor HIGH (black): Left motor stops, Right motor spins clockwise.
* Right sensor HIGH (black): Left motor spins clockwise, Right motor stops.
* Both sensors HIGH (black): Both motors stop.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `leftState == WHITE_FLOOR && ...` | Matches when the robot is centered over the black line (line is in between the two sensors). |
| `robotTurnLeft()` | Stops the left wheel so the right wheel rotates the chassis left, back onto the track. |
| `delay(10)` | Ensures high-frequency polling so the robot doesn't overshoot sharp turns. |

## Hardware & Safety Concept: IR Reflective Sensors (Phototransistors)
IR line sensors consist of an infrared LED emitter and a phototransistor receiver. The LED emits invisible infrared light downward. If the surface is white, it reflects the light, activating the phototransistor (outputs LOW). If the surface is black, the dark color absorbs the infrared light, leaving the phototransistor inactive (outputs HIGH).

## Try This! (Challenges)
1. **Pivot steering turns**: Modify `robotTurnLeft()` to use active pivot turns (Left wheel backward, Right wheel forward) for sharper turn capabilities on loop tracks.
2. **Speed Scaling adjustment**: Connect a potentiometer (GPIO 34) that scales the base motor PWM speeds to adjust the line following pace.
3. **Loss of Track safety search**: If the robot reads white on both sensors for more than 2 seconds, stop the motors to prevent runaway.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot turns away from the line | Steering pins swapped | Swap the `robotTurnLeft` and `robotTurnRight` function calls |
| Robot overshoots turns | Speed is too fast or scan rate too slow | Lower motor speeds or decrease the loop delay |
| Sensors do not trigger | Sensor height incorrect or dirty lens | Keep sensor pads 5–10 mm above the surface |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [122 - ESP32 Mobile Robot Left/Right Steering](122-esp32-mobile-robot-left-right-steering.md)
- [37 - ESP32 IR Obstacle Sensor Print](../beginner/37-esp32-ir-obstacle-sensor-print.md)
- [124 - ESP32 Obstacle Avoidance Robot](124-esp32-obstacle-avoidance-robot.md)
