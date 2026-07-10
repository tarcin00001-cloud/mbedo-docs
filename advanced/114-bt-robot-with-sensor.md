# 114 - BT Robot with Sensor

Combine Bluetooth remote control with autonomous obstacle sensing: a BT command drives the robot normally, but an HC-SR04 sensor overrides the command and halts the robot automatically when an obstacle is closer than 25 cm.

## Goal
Learn how to fuse two independent input sources — a wireless Bluetooth command channel and a proximity sensor — into a single motor control decision loop, prioritising safety over operator commands.

## What You Will Build
The robot accepts characters via the HC-05 Bluetooth module:
- **'F'**: Drive forward (if no obstacle detected).
- **'B'**: Drive backward.
- **'L'**: Pivot left.
- **'R'**: Pivot right.
- **'S'**: Stop.

**Sensor override**: On every loop the sketch measures distance. If the robot is commanded forward ('F') but the HC-SR04 detects an obstacle closer than 25 cm, the forward command is automatically overridden and the motors stop. All other commands (backward, turn, stop) work regardless of sensor distance, so the operator can still reverse away from obstacles.

**Why D2, D3 (BT), D4–D7 (L298N direction), D9, D10 (PWM speed)?** SoftwareSerial uses D2 (RX from BT TX) and D3 (TX to BT RX). L298N direction pins occupy D4–D7. Enable PWM pins use D9 and D10. The HC-SR04 uses D11 (TRIG) and D12 (ECHO) to avoid conflicts.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-05 Bluetooth Module | `hc05_bluetooth` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `hc_sr04` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| DC Motor (Left) | `dc_motor` | Yes | Yes |
| DC Motor (Right) | `dc_motor` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-05 Module | TXD | D2 | SoftwareSerial RX |
| HC-05 Module | RXD | D3 | SoftwareSerial TX (use voltage divider on hardware) |
| HC-05 Module | VCC | 5V | Power supply |
| HC-05 Module | GND | GND | Ground reference |
| HC-SR04 | VCC | 5V | Power supply |
| HC-SR04 | TRIG | D11 | Trigger pulse output |
| HC-SR04 | ECHO | D12 | Echo pulse input |
| HC-SR04 | GND | GND | Ground reference |
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
#include <SoftwareSerial.h>

SoftwareSerial BT(2, 3); // RX = D2, TX = D3

const int TRIG_PIN = 11;
const int ECHO_PIN = 12;

const int ENA = 9;
const int IN1 = 4;
const int IN2 = 5;

const int IN3 = 6;
const int IN4 = 7;
const int ENB = 10;

const int DRIVE_SPEED = 200;
const int TURN_SPEED  = 160;
const int STOP_DIST   = 25; // cm

char lastCmd = 'S';

void setup() {
  Serial.begin(9600);
  BT.begin(9600);

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  pinMode(ENA, OUTPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  pinMode(ENB, OUTPUT);

  // Start stopped
  digitalWrite(IN1, LOW); digitalWrite(IN2, LOW); analogWrite(ENA, 0);
  digitalWrite(IN3, LOW); digitalWrite(IN4, LOW); analogWrite(ENB, 0);

  Serial.println("BT Robot with Sensor Ready");
  Serial.println("Commands: F=Forward B=Backward L=Left R=Right S=Stop");
}

void loop() {
  // Read BT command if available
  if (BT.available() > 0) {
    char cmd = BT.read();
    if (cmd == 'F' || cmd == 'f' ||
        cmd == 'B' || cmd == 'b' ||
        cmd == 'L' || cmd == 'l' ||
        cmd == 'R' || cmd == 'r' ||
        cmd == 'S' || cmd == 's') {
      lastCmd = cmd;
      Serial.print("BT Command: "); Serial.println(lastCmd);
    }
  }

  // Measure distance
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  long duration = pulseIn(ECHO_PIN, HIGH);
  long dist = duration * 0.034 / 2;

  // Sensor override: block forward if obstacle detected
  bool obstacleAhead = (dist > 0 && dist < STOP_DIST);

  if (obstacleAhead && (lastCmd == 'F' || lastCmd == 'f')) {
    Serial.print("OBSTACLE at "); Serial.print(dist); Serial.println(" cm — overriding FORWARD: STOP");
    digitalWrite(IN1, LOW); digitalWrite(IN2, LOW); analogWrite(ENA, 0);
    digitalWrite(IN3, LOW); digitalWrite(IN4, LOW); analogWrite(ENB, 0);
    return;
  }

  // Execute BT command
  if (lastCmd == 'F' || lastCmd == 'f') {
    digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW); analogWrite(ENA, DRIVE_SPEED);
    digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW); analogWrite(ENB, DRIVE_SPEED);

  } else if (lastCmd == 'B' || lastCmd == 'b') {
    digitalWrite(IN1, LOW); digitalWrite(IN2, HIGH); analogWrite(ENA, DRIVE_SPEED);
    digitalWrite(IN3, LOW); digitalWrite(IN4, HIGH); analogWrite(ENB, DRIVE_SPEED);

  } else if (lastCmd == 'L' || lastCmd == 'l') {
    // Pivot left: reverse left, forward right
    digitalWrite(IN1, LOW); digitalWrite(IN2, HIGH); analogWrite(ENA, TURN_SPEED);
    digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW); analogWrite(ENB, TURN_SPEED);

  } else if (lastCmd == 'R' || lastCmd == 'r') {
    // Pivot right: forward left, reverse right
    digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW); analogWrite(ENA, TURN_SPEED);
    digitalWrite(IN3, LOW); digitalWrite(IN4, HIGH); analogWrite(ENB, TURN_SPEED);

  } else {
    // Stop
    digitalWrite(IN1, LOW); digitalWrite(IN2, LOW); analogWrite(ENA, 0);
    digitalWrite(IN3, LOW); digitalWrite(IN4, LOW); analogWrite(ENB, 0);
  }

  delay(80);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **HC-05 Bluetooth**, **HC-SR04**, **L298N**, and **two DC Motors** onto the canvas.
2. Wire HC-05: **TXD** → **D2**, **RXD** → **D3**, **VCC** → **5V**, **GND** → **GND**.
3. Wire HC-SR04: **TRIG** → **D11**, **ECHO** → **D12**, **VCC** → **5V**, **GND** → **GND**.
4. Wire L298N: **ENA** → **D9**, **IN1** → **D4**, **IN2** → **D5**, **IN3** → **D6**, **IN4** → **D7**, **ENB** → **D10**, **VIN** → **5V**, **GND** → **GND**.
5. Connect Left Motor to **OUT1/OUT2** and Right Motor to **OUT3/OUT4**.
6. Paste the code into the editor.
7. Use the default Arduino interpreted runtime.
8. Click **Run**.
9. Open the BT terminal panel and send `F`, `B`, `L`, `R`, `S` characters. Reduce the HC-SR04 distance below 25 cm while sending `F` to trigger the sensor override.

## Expected Output

Terminal:
```
BT Robot with Sensor Ready
Commands: F=Forward B=Backward L=Left R=Right S=Stop
BT Command: F
BT Command: F
OBSTACLE at 18 cm — overriding FORWARD: STOP
BT Command: B
BT Command: S
```

## Expected Canvas Behavior
| BT Command | HC-SR04 Distance | Action | Left Motor | Right Motor |
| --- | --- | --- | --- | --- |
| F | >= 25 cm (clear) | Forward | Full CW | Full CW |
| F | < 25 cm (obstacle) | Sensor override — STOP | Off | Off |
| B | any | Backward | CCW | CCW |
| L | any | Pivot Left | CCW | CW |
| R | any | Pivot Right | CW | CCW |
| S | any | Stop | Off | Off |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `char lastCmd = 'S'` | Stores the most recent valid command so the motors continue executing it on subsequent loops even if no new BT data arrives. |
| `BT.available() > 0` | Checks the SoftwareSerial RX buffer; returns true only when a new byte has arrived from the HC-05. |
| `obstacleAhead && (lastCmd == 'F')` | The safety interlock: sensor proximity only overrides when the robot is commanded to move toward the obstacle. |
| `return` inside the override block | Skips the rest of the loop body immediately after stopping, preventing any motor-on commands from executing. |
| `dist > 0 && dist < STOP_DIST` | Guards against false 0-cm readings from the HC-SR04 dead zone that would lock out forward motion permanently. |

## Hardware & Safety Concept: Sensor-Override Priority
In autonomous and semi-autonomous vehicles, sensors that detect immediate physical danger (collision, cliff, overheat) are always given higher priority than remote operator commands. This is called **safety interlock** design. The BT command tells the robot what the operator wants; the sensor override tells the robot what physics allows. The sensor wins because the speed of light (and electricity) always beats human reaction time.

## Try This! (Challenges)
1. **Audible Obstacle Warning**: Connect a buzzer to D8. When `obstacleAhead` is true, call `tone(8, 1000, 200)` to beep once per loop while the override is active.
2. **Distance Reporting**: After each BT command is accepted, reply back via `BT.print()` with the current measured distance so the remote operator can see proximity data on their phone.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot ignores BT commands | SoftwareSerial pins wrong | Confirm HC-05 TX connects to Arduino D2 (RX) and HC-05 RX connects to Arduino D3 (TX). |
| Obstacle override never triggers | Sensor wiring error | Verify TRIG is D11 (OUTPUT) and ECHO is D12 (INPUT); check the `pulseIn` return is non-zero. |
| Robot drives forward even with obstacle | Uppercase/lowercase mismatch | The `lastCmd == 'F'` check already handles both 'F' and 'f' — ensure no other command is stored in `lastCmd`. |

## Mode Notes
`SoftwareSerial`, `BT.available()`, `BT.read()`, `pulseIn`, `analogWrite`, and `digitalWrite` are all supported by MbedO interpreted mode.

## Related Projects
- [85 - BT Remote Car](../intermediate/85-bt-remote-car.md)
- [77 - BT Distance Report](../intermediate/77-bt-distance-report.md)
- [112 - Obstacle Avoider](112-obstacle-avoider.md)
- [113 - Line Follower](113-line-follower.md)
