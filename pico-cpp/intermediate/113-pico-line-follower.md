# 113 - Pico Line Follower

Build an autonomous line-following robot using two infrared reflective track sensors and dual DC motors.

## Goal
Learn how to implement classic differential steering logic based on digital infrared track sensor inputs.

## What You Will Build
An autonomous line-following robot:
- **Left IR Sensor (GP16)**: Detects the dark line (Active-HIGH or Active-LOW depending on model).
- **Right IR Sensor (GP17)**: Detects the dark line.
- **L298N & Dual DC Motors (GP10-15)**: Steers the robot forward or pivots left/right to keep the chassis centered over the track line.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| IR Reflective Sensors | `button` | Yes (represented by two switches) | Yes (TCRT5000 modules) |
| L298N Motor Driver | `l298n` | Yes | Yes |
| DC Motors | `dc_motor` | Yes (two motors) | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Left IR Sensor | OUT | GP16 | Left track monitor |
| Right IR Sensor | OUT | GP17 | Right track monitor |
| Both IR Sensors | VCC | 3.3V | Sensor power |
| Both IR Sensors | GND | GND | Shared ground |
| L298N Control | ENA / ENB | GP10 / GP11 | Left/Right motor speed |
| L298N Control | IN1 / IN2 | GP12 / GP13 | Left motor direction |
| L298N Control | IN3 / IN4 | GP14 / GP15 | Right motor direction |
| L298N | GND | GND | Ground return |

## Code
```cpp
const int LEFT_IR  = 16;
const int RIGHT_IR = 17;

const int ENA = 10;
const int ENB = 11;
const int IN1 = 12;
const int IN2 = 13;
const int IN3 = 14;
const int IN4 = 15;

void setup() {
  pinMode(LEFT_IR, INPUT);
  pinMode(RIGHT_IR, INPUT);
  
  pinMode(ENA, OUTPUT);
  pinMode(ENB, OUTPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  // Set motor speed pins HIGH (full speed)
  digitalWrite(ENA, HIGH);
  digitalWrite(ENB, HIGH);
}

void loop() {
  // Read infrared sensor states
  // TCRT5000 outputs LOW on reflective light (white surface), HIGH on dark (black line)
  int leftState  = digitalRead(LEFT_IR);
  int rightState = digitalRead(RIGHT_IR);

  // 1. Both sensors on white surface: move straight forward
  if (leftState == LOW && rightState == LOW) {
    moveForward();
  }
  // 2. Left sensor hits black line: turn left to adjust
  else if (leftState == HIGH && rightState == LOW) {
    turnLeft();
  }
  // 3. Right sensor hits black line: turn right to adjust
  else if (leftState == LOW && rightState == HIGH) {
    turnRight();
  }
  // 4. Both sensors hit black line: stop robot
  else if (leftState == HIGH && rightState == HIGH) {
    stopRobot();
  }

  delay(20); // High correction rate
}

// Direction control helpers
void moveForward() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
}

void turnLeft() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, LOW);  // Stop left wheel
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  // Drive right wheel forward
}

void turnRight() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  // Drive left wheel forward
  digitalWrite(IN3, LOW);  digitalWrite(IN4, LOW);  // Stop right wheel
}

void stopRobot() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);  digitalWrite(IN4, LOW);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two IR sensors** (represented by switch buttons), **L298N**, and **two DC Motors** onto the canvas.
2. Connect Left IR OUT to **GP16**, Right IR OUT to **GP17**. Connect L298N to **GP10-15**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Click the switch buttons representing the sensors to simulate the robot hitting a black line, and watch the wheels steer.

## Expected Output

Terminal:
```
Simulation active. Line follower tracking active.
```

## Expected Canvas Behavior
| Left Sensor Switch | Right Sensor Switch | Left Wheel | Right Wheel | Robot Action |
| --- | --- | --- | --- | --- |
| Open (LOW) | Open (LOW) | Forward | Forward | Straight Forward |
| Closed (HIGH) | Open (LOW) | Stopped | Forward | Turn Left |
| Open (LOW) | Closed (HIGH) | Forward | Stopped | Turn Right |
| Closed (HIGH) | Closed (HIGH) | Stopped | Stopped | Stop |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `leftState == HIGH && rightState == LOW` | Evaluates if only the left sensor has drifted over the dark line, calling the correction steer. |

## Hardware & Safety Concept: Reflective Sensor Calibration
TCRT5000 infrared reflective sensors contain an infrared LED transmitter and a phototransistor receiver. The sensor reads the intensity of the bounced infrared light. Because black surfaces absorb light and white surfaces reflect it, the output voltage shifts. Real sensor modules include a small **trimmer potentiometer** that must be manually calibrated using a screwdriver to set the exact digital switching point between your floor and your black track tape.

## Try This! (Challenges)
1. **Dynamic Speed control**: Use `analogWrite` to slow down the wheels when turning to make the tracking smoother.
2. **Reverse search**: Add code so that if both sensors lose the line (e.g. read LOW for more than 1 second), the robot reverses slowly to find the track.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot turns away from the line rather than towards it | Sensor wires swapped | Swap the GP16 and GP17 input pin definitions in your code, or swap the physical sensor wires. |

## Mode Notes
This multi-device control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [37 - Pico IR Obstacle](../../beginner/37-pico-ir-obstacle.md)
- [112 - Pico Bluetooth Robot](112-pico-bt-robot.md)
