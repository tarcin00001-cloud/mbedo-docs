# 112 - Obstacle Avoider

Build an autonomous robot that measures distance with an HC-SR04 ultrasonic sensor and uses an L298N motor driver to stop and reverse when an obstacle is closer than 20 cm, then drive forward again when the path is clear.

## Goal
Learn how to combine a distance sensor with an H-bridge motor driver to implement simple reactive obstacle avoidance — the foundation of all autonomous ground vehicles.

## What You Will Build
The sketch reads distance from the HC-SR04 every loop iteration and applies a threshold rule:

- **Distance < 20 cm (obstacle)**: Both motors stop immediately, then reverse for 600 ms to move away from the obstacle.
- **Distance >= 20 cm (clear)**: Both motors drive forward at full speed.

The current distance and action print to the Serial Terminal continuously so you can watch the decision-making in real time.

**Why D2–D7 and D9/D10?** D2 and D3 are the ultrasonic TRIG and ECHO pins. D4–D7 are the L298N direction pins (IN1–IN4). D9 and D10 are the PWM enable channels (ENA, ENB) that set motor speed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `hc_sr04` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| DC Motor (Left) | `dc_motor` | Yes | Yes |
| DC Motor (Right) | `dc_motor` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-SR04 | VCC | 5V | Power supply |
| HC-SR04 | TRIG | D2 | Trigger pulse output |
| HC-SR04 | ECHO | D3 | Echo pulse input |
| HC-SR04 | GND | GND | Ground reference |
| L298N | ENA | D9 | Left motor speed (PWM) |
| L298N | IN1 | D4 | Left motor direction 1 |
| L298N | IN2 | D5 | Left motor direction 2 |
| L298N | IN3 | D6 | Right motor direction 1 |
| L298N | IN4 | D7 | Right motor direction 2 |
| L298N | ENB | D10 | Right motor speed (PWM) |
| L298N | VIN | 5V | Power supply |
| L298N | GND | GND | Ground reference |
| Left Motor | +/- | OUT1/OUT2 | Left channel output |
| Right Motor | +/- | OUT3/OUT4 | Right channel output |

## Code
```cpp
const int TRIG_PIN = 2;
const int ECHO_PIN = 3;

const int ENA = 9;
const int IN1 = 4;
const int IN2 = 5;

const int IN3 = 6;
const int IN4 = 7;
const int ENB = 10;

const int OBSTACLE_CM = 20;
const int DRIVE_SPEED = 200;

void setup() {
  Serial.begin(9600);

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  pinMode(ENA, OUTPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  pinMode(ENB, OUTPUT);

  // Start with motors stopped
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  analogWrite(ENA, 0);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
  analogWrite(ENB, 0);

  Serial.println("Obstacle Avoider Ready");
}

void loop() {
  // Measure distance with HC-SR04
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH);
  long dist = duration * 0.034 / 2;

  Serial.print("Distance: ");
  Serial.print(dist);
  Serial.print(" cm  -> ");

  if (dist > 0 && dist < OBSTACLE_CM) {
    // OBSTACLE — stop then reverse
    Serial.println("OBSTACLE: Stop + Reverse");

    // Stop
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
    analogWrite(ENA, 0);
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, LOW);
    analogWrite(ENB, 0);
    delay(200);

    // Reverse
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, HIGH);
    analogWrite(ENA, DRIVE_SPEED);
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, HIGH);
    analogWrite(ENB, DRIVE_SPEED);
    delay(600);

    // Stop before re-evaluating
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
    analogWrite(ENA, 0);
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, LOW);
    analogWrite(ENB, 0);
    delay(200);

  } else {
    // CLEAR — drive forward
    Serial.println("CLEAR: Forward");

    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
    analogWrite(ENA, DRIVE_SPEED);
    digitalWrite(IN3, HIGH);
    digitalWrite(IN4, LOW);
    analogWrite(ENB, DRIVE_SPEED);
  }

  delay(100);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **HC-SR04**, **L298N**, and **two DC Motors** onto the canvas.
2. Connect HC-SR04: **VCC** → **5V**, **TRIG** → **D2**, **ECHO** → **D3**, **GND** → **GND**.
3. Connect L298N: **ENA** → **D9**, **IN1** → **D4**, **IN2** → **D5**, **IN3** → **D6**, **IN4** → **D7**, **ENB** → **D10**, **VIN** → **5V**, **GND** → **GND**.
4. Connect Left Motor to **OUT1/OUT2** and Right Motor to **OUT3/OUT4**.
5. Paste the code into the editor.
6. Use the default Arduino interpreted runtime.
7. Click **Run**.
8. Adjust the HC-SR04 distance slider below 20 cm to trigger obstacle avoidance, then raise it above 20 cm to resume forward motion.

## Expected Output

Terminal:
```
Obstacle Avoider Ready
Distance: 45 cm  -> CLEAR: Forward
Distance: 45 cm  -> CLEAR: Forward
Distance: 12 cm  -> OBSTACLE: Stop + Reverse
Distance: 30 cm  -> CLEAR: Forward
```

## Expected Canvas Behavior
| HC-SR04 Distance | Action | Left Motor | Right Motor |
| --- | --- | --- | --- |
| >= 20 cm | Forward | Clockwise | Clockwise |
| < 20 cm | Stop (200 ms) | Off | Off |
| < 20 cm (reverse phase) | Reverse (600 ms) | Counter-CW | Counter-CW |
| Re-measured >= 20 cm | Forward | Clockwise | Clockwise |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `duration * 0.034 / 2` | Converts the echo pulse duration in microseconds to centimeters (speed of sound 340 m/s ÷ 2 for round trip). |
| `dist > 0 && dist < OBSTACLE_CM` | Guards against readings of 0 (no echo / dead zone) which would falsely trigger avoidance on every loop. |
| `digitalWrite(IN1, LOW); digitalWrite(IN2, LOW)` | Sets both motor direction pins LOW to coast-stop the motor before applying the reverse polarity. |
| `analogWrite(ENA, DRIVE_SPEED)` | Sets motor speed via PWM duty cycle; 200/255 ≈ 78% — fast enough to detect easily, slow enough to be safe. |
| `delay(200)` between stop and reverse | Adds a brief pause to let inertia dissipate before reversing direction, protecting the motor driver from back-EMF spikes. |

## Hardware & Safety Concept: H-Bridge Back-EMF Protection
When a DC motor is suddenly reversed, the spinning armature generates a brief high-voltage spike (back-EMF) in the opposite direction. The L298N includes internal flyback diodes that clamp this voltage and protect the driver IC. The short `delay(200)` coast-stop before reversing reduces the magnitude of this spike and extends driver lifetime on real hardware.

## Try This! (Challenges)
1. **Turn on Obstacle**: Instead of reversing straight back, set IN1/IN2 to reverse the left motor only, and IN3/IN4 to continue the right motor forward for 400 ms — this pivots the robot to avoid the obstacle at an angle.
2. **Speed from Sensor**: Use `map(dist, 20, 200, 80, 220)` to set motor speed proportionally to distance — faster when far away, slower when approaching an object.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot always shows OBSTACLE | ECHO/TRIG pins swapped | TRIG must be OUTPUT on D2, ECHO must be INPUT on D3. |
| Motors spin opposite direction | Motor wires reversed at L298N | Swap OUT1/OUT2 wires or swap IN1/IN2 logic in code. |
| Distance always reads 0 | Missing `pulseIn` support | Confirm MbedO version supports `pulseIn`; try adjusting the TRIG pulse `delayMicroseconds` to 10. |

## Mode Notes
`pulseIn`, `analogWrite`, `digitalWrite`, and `delay` are all supported by MbedO interpreted mode.

## Related Projects
- [44 - Obstacle Beeper](../intermediate/44-obstacle-beeper.md)
- [54 - Distance Serial](../intermediate/54-distance-serial.md)
- [84 - Robot Forward Back](../intermediate/84-robot-forward-back.md)
- [113 - Line Follower](113-line-follower.md)
- [114 - BT Robot with Sensor](114-bt-robot-with-sensor.md)
