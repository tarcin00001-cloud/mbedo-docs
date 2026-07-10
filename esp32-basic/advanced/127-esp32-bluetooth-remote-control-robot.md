# 127 - ESP32 Bluetooth Remote Control Robot

Build a wirelessly controlled mobile robot that receives motion commands ('F', 'B', 'L', 'R', 'S') from a Bluetooth terminal and drives the wheels accordingly.

## Goal
Learn how to parse serial character commands over a wireless Bluetooth connection to steer a differential-drive robot chassis.

## What You Will Build
An HC-05 Bluetooth module on UART2 (RX2: GPIO 16, TX2: GPIO 17) receives steer commands. An L298N driver controls two DC motors (Left: 18/19/5, Right: 21/22/23). Characters 'F' (Forward), 'B' (Backward), 'L' (Left), 'R' (Right), and 'S' (Stop) trigger corresponding movement states.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motors (2) | `dc_motor` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth_hc05` | Yes | Yes |
| External Power Supply (6–12 V) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Module | IN1 / IN2 / ENA | GPIO18 / GPIO19 / GPIO5 | Yellow/Green/Orange | Left Motor |
| L298N Module | IN3 / IN4 / ENB | GPIO21 / GPIO22 / GPIO23 | Blue/Purple/White | Right Motor |
| HC-05 Module | VCC / GND | 5V / GND | Red / Black | Power rails |
| HC-05 Module | TXD | GPIO16 (RX2) | Green | Bluetooth Transmit |
| HC-05 Module | RXD | GPIO17 (TX2) | Yellow | Bluetooth Receive |

> **Wiring tip:** Share the 5V Vin rail to power both the HC-05 module and the L298N logic. Make sure the battery power GND is tied to the ESP32 GND.

## Code
```cpp
// Bluetooth Remote Control Robot
#define RX2_PIN 16
#define TX2_PIN 17

const int IN1 = 18;
const int IN2 = 19;
const int ENA = 5;

const int IN3 = 21;
const int IN4 = 22;
const int ENB = 23;

void robotForward() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  digitalWrite(ENA, HIGH);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  digitalWrite(ENB, HIGH);
}

void robotReverse() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, HIGH); digitalWrite(ENA, HIGH);
  digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH); digitalWrite(ENB, HIGH);
}

void robotPivotLeft() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, HIGH); digitalWrite(ENA, HIGH);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  digitalWrite(ENB, HIGH);
}

void robotPivotRight() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  digitalWrite(ENA, HIGH);
  digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH); digitalWrite(ENB, HIGH);
}

void robotStop() {
  digitalWrite(ENA, LOW);
  digitalWrite(ENB, LOW);
}

void setup() {
  Serial.begin(115200);
  Serial2.begin(9600, SERIAL_8N1, RX2_PIN, TX2_PIN);
  
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT); pinMode(ENA, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT); pinMode(ENB, OUTPUT);
  
  robotStop(); // Start stopped
  Serial.println("Bluetooth Robot online. Ready for commands...");
}

void loop() {
  if (Serial2.available() > 0) {
    char cmd = (char)Serial2.read();
    Serial.print("BT Command: "); Serial.println(cmd);
    
    switch (cmd) {
      case 'F':
      case 'f':
        robotForward();
        Serial2.println("MOVING: FORWARD");
        break;
      case 'B':
      case 'b':
        robotReverse();
        Serial2.println("MOVING: REVERSE");
        break;
      case 'L':
      case 'l':
        robotPivotLeft();
        Serial2.println("STEERING: LEFT");
        break;
      case 'R':
      case 'r':
        robotPivotRight();
        Serial2.println("STEERING: RIGHT");
        break;
      case 'S':
      case 's':
        robotStop();
        Serial2.println("MOVING: STOPPED");
        break;
    }
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N**, two **DC Motors**, and **HC-05 Module** onto the canvas.
2. Wire HC-05 to **GPIO16/GPIO17** and L298N to **GPIO18, 19, 5, 21, 22, 23**.
3. Paste the code and click **Run**.
4. In the virtual Bluetooth text console, type `F` to drive forward, `L` to spin left, and `S` to stop. Watch the wheels rotate.

## Expected Output
Serial Monitor:
```
Bluetooth Robot online. Ready for commands...
BT Command: F
BT Command: L
BT Command: S
```

Bluetooth Terminal:
```
MOVING: FORWARD
STEERING: LEFT
MOVING: STOPPED
```

## Expected Canvas Behavior
* Sending `F` spins both motors clockwise.
* Sending `L` spins the left motor counter-clockwise and the right motor clockwise.
* Sending `S` stops both motors.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `Serial2.read()` | Checks the next available byte from the Bluetooth RX buffer. |
| `switch (cmd)` | Maps character commands instantly to steering functions. |
| `Serial2.println(...)` | Transmits status reports back to the mobile controller. |

## Hardware & Safety Concept: Command Loss-of-Signal Failsafes
In physical robotic applications, if the Bluetooth link drops (out of range or phone battery dies) while the robot is driving forward, the robot will keep driving indefinitely (a runaway robot). To prevent this, safety loops track the timestamp of the last received command (`millis()`). If no new command is received within 1 second, the robot is automatically halted.

## Try This! (Challenges)
1. **Heartbeat Loss-of-Signal Safety**: Implement a safety timer that calls `robotStop()` if no command is received for more than 1.5 seconds.
2. **Dynamic Speed scaling**: Add commands '1' to '5' that adjust the motor PWM speed percentage.
3. **Beep Alarm Headlights**: Add a buzzer on GPIO 15 and flash LEDs (headlights) on remote commands.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot does not steer correctly | Motor terminals reversed | Swap OUT1/OUT2 or OUT3/OUT4 wires at the driver terminals |
| Commands trigger twice | Carriage returns sent | Standard character parsing ignores `\n` in this switch-case; ensure no extra spaces are evaluated |
| Bluetooth drops out during motor start | Current sag | Power motors from a dedicated battery pack; share only GND |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [122 - ESP32 Mobile Robot Left/Right Steering](122-esp32-mobile-robot-left-right-steering.md)
- [89 - ESP32 HC-05 Bluetooth Serial UART Link](../intermediate/89-esp32-hc05-bluetooth-serial-uart-link.md)
- [128 - ESP32 Bluetooth Robot with Crash Override](128-esp32-bluetooth-robot-with-crash-override.md)
