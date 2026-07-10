# 113 - Line Follower

Use two digital line sensors to steer a two-wheeled robot along a dark line on a light surface, adjusting left and right motor speeds in real time based on sensor readings.

## Goal
Learn how line sensor digitalRead values can drive differential motor control, understanding the feedback loop that keeps a robot aligned to a path without any complex algorithms.

## What You Will Build
Two line sensors (left and right) are mounted under the chassis, pointing downward. Each returns LOW when it detects a dark line and HIGH when it sees the light surface.

The robot reacts to four possible sensor states:

| Left Sensor | Right Sensor | Action |
| --- | --- | --- |
| LOW (on line) | LOW (on line) | Drive forward |
| LOW (on line) | HIGH (off line) | Turn right (slow left motor) |
| HIGH (off line) | LOW (on line) | Turn left (slow right motor) |
| HIGH (off line) | HIGH (off line) | Stop and search (slow reverse) |

**Why D2, D3, D4–D7, D9, D10?** D2 and D3 are digital inputs for the left and right line sensors. D4–D7 are the L298N direction pins. D9 and D10 are PWM enable channels for motor speed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Line Sensor (Left) | `line_sensor` | Yes | Yes |
| Line Sensor (Right) | `line_sensor` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| DC Motor (Left) | `dc_motor` | Yes | Yes |
| DC Motor (Right) | `dc_motor` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Left Line Sensor | VCC | 5V | Power supply |
| Left Line Sensor | OUT | D2 | Digital line detect signal |
| Left Line Sensor | GND | GND | Ground reference |
| Right Line Sensor | VCC | 5V | Power supply |
| Right Line Sensor | OUT | D3 | Digital line detect signal |
| Right Line Sensor | GND | GND | Ground reference |
| L298N | ENA | D9 | Left motor speed (PWM) |
| L298N | IN1 | D4 | Left motor direction 1 |
| L298N | IN2 | D5 | Left motor direction 2 |
| L298N | IN3 | D6 | Right motor direction 1 |
| L298N | IN4 | D7 | Right motor direction 2 |
| L298N | ENB | D10 | Right motor speed (PWM) |
| L298N | VIN | 5V | Power supply |
| L298N | GND | GND | Ground reference |
| Left Motor | +/- | OUT1/OUT2 | Left channel |
| Right Motor | +/- | OUT3/OUT4 | Right channel |

## Code
```cpp
const int LEFT_SENSOR  = 2;
const int RIGHT_SENSOR = 3;

const int ENA = 9;
const int IN1 = 4;
const int IN2 = 5;

const int IN3 = 6;
const int IN4 = 7;
const int ENB = 10;

const int FULL_SPEED = 200;
const int TURN_SPEED = 130;

void setup() {
  Serial.begin(9600);

  pinMode(LEFT_SENSOR,  INPUT);
  pinMode(RIGHT_SENSOR, INPUT);

  pinMode(ENA, OUTPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  pinMode(ENB, OUTPUT);

  // Start stopped
  digitalWrite(IN1, LOW); digitalWrite(IN2, LOW); analogWrite(ENA, 0);
  digitalWrite(IN3, LOW); digitalWrite(IN4, LOW); analogWrite(ENB, 0);

  Serial.println("Line Follower Ready");
}

void loop() {
  int leftVal  = digitalRead(LEFT_SENSOR);
  int rightVal = digitalRead(RIGHT_SENSOR);

  Serial.print("L:"); Serial.print(leftVal);
  Serial.print(" R:"); Serial.print(rightVal);
  Serial.print("  -> ");

  if (leftVal == LOW && rightVal == LOW) {
    // Both on line — go forward
    Serial.println("FORWARD");
    digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW); analogWrite(ENA, FULL_SPEED);
    digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW); analogWrite(ENB, FULL_SPEED);

  } else if (leftVal == LOW && rightVal == HIGH) {
    // Left on line, right off — steer right (slow left, fast right)
    Serial.println("STEER RIGHT");
    digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW); analogWrite(ENA, TURN_SPEED);
    digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW); analogWrite(ENB, FULL_SPEED);

  } else if (leftVal == HIGH && rightVal == LOW) {
    // Right on line, left off — steer left (fast left, slow right)
    Serial.println("STEER LEFT");
    digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW); analogWrite(ENA, FULL_SPEED);
    digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW); analogWrite(ENB, TURN_SPEED);

  } else {
    // Both off line — stop and search (slow reverse)
    Serial.println("LOST — searching");
    digitalWrite(IN1, LOW); digitalWrite(IN2, HIGH); analogWrite(ENA, TURN_SPEED);
    digitalWrite(IN3, LOW); digitalWrite(IN4, HIGH); analogWrite(ENB, TURN_SPEED);
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **two Line Sensors**, **L298N**, and **two DC Motors** onto the canvas.
2. Connect Left Line Sensor: **VCC** → **5V**, **OUT** → **D2**, **GND** → **GND**.
3. Connect Right Line Sensor: **VCC** → **5V**, **OUT** → **D3**, **GND** → **GND**.
4. Connect L298N: **ENA** → **D9**, **IN1** → **D4**, **IN2** → **D5**, **IN3** → **D6**, **IN4** → **D7**, **ENB** → **D10**, **VIN** → **5V**, **GND** → **GND**.
5. Connect Left Motor to **OUT1/OUT2** and Right Motor to **OUT3/OUT4**.
6. Paste the code into the editor.
7. Use the default Arduino interpreted runtime.
8. Click **Run**.
9. Toggle the Left and Right sensor buttons on the canvas to simulate the robot moving on and off the line, and watch the motor directions change in the Terminal.

## Expected Output

Terminal:
```
Line Follower Ready
L:0 R:0  -> FORWARD
L:0 R:1  -> STEER RIGHT
L:1 R:0  -> STEER LEFT
L:1 R:1  -> LOST — searching
```

## Expected Canvas Behavior
| Left Sensor | Right Sensor | Action | Left Motor | Right Motor |
| --- | --- | --- | --- | --- |
| LOW (0) | LOW (0) | FORWARD | Full CW | Full CW |
| LOW (0) | HIGH (1) | STEER RIGHT | Slow CW | Full CW |
| HIGH (1) | LOW (0) | STEER LEFT | Full CW | Slow CW |
| HIGH (1) | HIGH (1) | LOST — searching | Slow CCW | Slow CCW |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalRead(LEFT_SENSOR)` | Returns LOW (0) when the sensor detects a dark line beneath it, HIGH (1) on a light surface. |
| `analogWrite(ENA, TURN_SPEED)` | Reduces the left motor speed to 130/255 ≈ 51%, causing a gradual turn instead of a sharp pivot. |
| `analogWrite(ENA, FULL_SPEED)` | Runs the motor at 200/255 ≈ 78% to allow margin for correction PWM while staying fast. |
| `else` (both HIGH) reverse | Slowly reverses both motors when the line is completely lost, giving the sensors a chance to re-detect it. |

## Hardware & Safety Concept: Proportional vs. Bang-Bang Control
This sketch uses **bang-bang control** — the motor is either at FULL_SPEED or TURN_SPEED with no gradual ramp. Bang-bang is simple to implement but causes oscillation (the robot weaves). A **proportional controller** would continuously vary the speed difference between motors in proportion to how far off-center the robot is, giving smoother tracking. As a next step, try using `analogRead` with an analog line sensor and `map()` to compute a proportional correction.

## Try This! (Challenges)
1. **Pivot Turn**: Replace the steer logic with a full motor reversal on one side (`IN2 HIGH, IN1 LOW`) at `TURN_SPEED` to make the robot pivot sharply back onto the line instead of gradually curving.
2. **Speed Trim via Potentiometer**: Connect a potentiometer to A0 and use `analogRead(A0)` with `map()` to set `FULL_SPEED` dynamically, letting you tune the following speed without recompiling.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot always reads LOST | Sensors not outputting LOW | Check that sensor OUT is connected to D2/D3 as INPUT; verify sensor threshold pot is adjusted correctly. |
| Robot turns wrong direction | Left/Right sensor wires swapped | Swap the sensor connections at D2 and D3. |
| Robot spins in circles | IN1/IN2 wires swapped on L298N | Swap the wires on one motor channel at the L298N output terminals. |

## Mode Notes
`digitalRead`, `analogWrite`, and `digitalWrite` in all four if/else branches are fully supported by MbedO interpreted mode.

## Related Projects
- [84 - Robot Forward Back](../intermediate/84-robot-forward-back.md)
- [112 - Obstacle Avoider](112-obstacle-avoider.md)
- [114 - BT Robot with Sensor](114-bt-robot-with-sensor.md)
