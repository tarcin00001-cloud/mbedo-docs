# 112 - Pico Bluetooth Robot

Build a two-wheel drive mobile robot chassis that is steered wirelessly using Bluetooth character commands.

## Goal
Learn how to parse steering command characters received over UART and actuate dual H-Bridge DC motor drivers.

## What You Will Build
A wireless steered robot chassis:
- **HC-05 Bluetooth Module (GP0, GP1)**: Receives commands.
- **L298N Motor Driver (GP10-15)**: Controls two DC motors.
- **Steering Commands**:
  - `'F'` (Forward)
  - `'B'` (Backward)
  - `'L'` (Left turn)
  - `'R'` (Right turn)
  - `'S'` (Stop)

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| DC Motors | `dc_motor` | Yes (two motors) | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HC-05 | TXD | GP1 (RX) | Data to Pico |
| HC-05 | RXD | GP0 (TX) | Data from Pico (with divider) |
| L298N | ENA | GP10 | Left motor speed control |
| L298N | ENB | GP11 | Right motor speed control |
| L298N | IN1 / IN2 | GP12 / GP13 | Left motor direction |
| L298N | IN3 / IN4 | GP14 / GP15 | Right motor direction |
| L298N | GND | GND | Shared ground |

## Code
```cpp
const int ENA = 10;
const int ENB = 11;
const int IN1 = 12;
const int IN2 = 13;
const int IN3 = 14;
const int IN4 = 15;

void setup() {
  Serial1.begin(9600);
  
  pinMode(ENA, OUTPUT);
  pinMode(ENB, OUTPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  // Set motor speed pins HIGH (full speed)
  digitalWrite(ENA, HIGH);
  digitalWrite(ENB, HIGH);
  
  stopRobot();
  Serial1.println("Bluetooth Robot Base Ready. Send F, B, L, R, or S");
}

void loop() {
  if (Serial1.available()) {
    char cmd = Serial1.read();

    if (cmd == 'F') {
      moveForward();
      Serial1.println("DIR: FORWARD");
    } 
    else if (cmd == 'B') {
      moveBackward();
      Serial1.println("DIR: BACKWARD");
    } 
    else if (cmd == 'L') {
      turnLeft();
      Serial1.println("DIR: LEFT");
    } 
    else if (cmd == 'R') {
      turnRight();
      Serial1.println("DIR: RIGHT");
    } 
    else if (cmd == 'S') {
      stopRobot();
      Serial1.println("DIR: STOPPED");
    }
  }
}

// Direction control helpers
void moveForward() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
}

void moveBackward() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH);
}

void turnLeft() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, HIGH); // Left backward
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  // Right forward
}

void turnRight() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  // Left forward
  digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH); // Right backward
}

void stopRobot() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);  digitalWrite(IN4, LOW);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-05 Bluetooth Module**, **L298N**, and **two DC Motors** onto the canvas.
2. Connect HC-05: **TXD** to **GP1**, **RXD** to **GP0**. Connect L298N control lines to **GP10-15**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Type `'F'` (Forward), `'L'` (Left), or `'S'` (Stop) in the Bluetooth terminal window to steer the robot.

## Expected Output

Terminal:
```
Bluetooth Robot Base Ready. Send F, B, L, R, or S
DIR: FORWARD
DIR: LEFT
DIR: STOPPED
```

## Expected Canvas Behavior
| Bluetooth Command | Left Motor | Right Motor | Robot Movement |
| --- | --- | --- | --- |
| `'F'` | Spin Forward | Spin Forward | Forward |
| `'L'` | Spin Reverse | Spin Forward | Left Pivot |
| `'S'` | Stopped | Stopped | Stopped |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(IN1, HIGH)` | Drives Left Motor Phase 1 high to spin the left side wheel forward. |

## Hardware & Safety Concept: Separate Power Rail Isolation
DC motors draw high currents and create electrical noise (inductive voltage spikes) that can cause microcontrollers to freeze or reset. To prevent this, always separate control power from motor power. The Pico should be powered by USB or a 5V supply, while the L298N motor power pins (12V and GND) connect directly to a separate motor battery pack. Ensure the grounds of the battery pack and Pico are tied together.

## Try This! (Challenges)
1. **Interactive Speed Adjustment**: Use `analogWrite(ENA, speed)` to make the robot steer at 50% speed when the command `'H'` (Half Speed) is received.
2. **Alert Headlights**: Add two headlights (LEDs) on GP16 and GP17 that light up when the robot moves forward.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot spins in circles when forward is sent | One motor's polarity is reversed | Swap the positive and negative wires of the circular motor that is spinning backward on the L298N output block. |

## Mode Notes
This multi-device control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [53 - Pico Motor Toggle](../intermediate/53-pico-motor-toggle.md)
- [55 - Pico Motor Direction](../intermediate/55-pico-motor-direction.md)
- [89 - Pico Bluetooth Serial](../intermediate/89-pico-bt-serial.md)
