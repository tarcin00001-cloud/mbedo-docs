# 121 - ESP32 Dual Motor Forward/Reverse Drive

Control a dual-motor mobile robot chassis through an L298N H-bridge motor driver, coordinating both left and right wheels to drive forward, stop, and reverse in sequence.

## Goal
Learn how to wire and program a dual H-bridge driver circuit to actuate two DC motors simultaneously, configuring direction bits and enable signals.

## What You Will Build
An L298N motor driver controls two DC motors (Left and Right). GPIOs 18, 19, and 5 control the Left Motor (IN1, IN2, ENA); GPIOs 21, 22, and 23 control the Right Motor (IN3, IN4, ENB). The robot drives forward for 3 seconds, stops for 1 second, reverses for 3 seconds, and stops.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motors (2) | `dc_motor` | Yes | Yes |
| External Power Supply (6–12 V) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Module | IN1 | GPIO18 | Yellow | Left Motor Direction 1 |
| L298N Module | IN2 | GPIO19 | Green | Left Motor Direction 2 |
| L298N Module | ENA | GPIO5 | Orange | Left Motor Enable |
| L298N Module | IN3 | GPIO21 | Blue | Right Motor Direction 1 |
| L298N Module | IN4 | GPIO22 | Purple | Right Motor Direction 2 |
| L298N Module | ENB | GPIO23 | White | Right Motor Enable |
| L298N Module | GND | GND | Black | Shared logic ground |
| L298N Module | 12V (VCC) | External battery + | Red | Motor power supply |
| L298N Module | GND (power) | External battery − | Black | Motor power ground |
| Left DC Motor | Terminals | OUT1 / OUT2 | Blue / White | Left wheel connections |
| Right DC Motor | Terminals | OUT3 / OUT4 | Blue / White | Right wheel connections |

> **Wiring tip:** When working with dual motors, make sure to remove both the ENA and ENB jumpers from the L298N board so that you can control each channel's enable state independently using GPIOs 5 and 23. Always connect the external battery ground to the ESP32 GND pin.

## Code
```cpp
// Dual Motor Forward/Reverse Drive
const int IN1 = 18; // Left Motor direction bit 1
const int IN2 = 19; // Left Motor direction bit 2
const int ENA = 5;  // Left Motor enable (speed/on-off)

const int IN3 = 21; // Right Motor direction bit 1
const int IN4 = 22; // Right Motor direction bit 2
const int ENB = 23; // Right Motor enable (speed/on-off)

void robotForward() {
  // Left Motor FWD
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(ENA, HIGH);
  
  // Right Motor FWD
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  digitalWrite(ENB, HIGH);
  
  Serial.println("Robot: FORWARD");
}

void robotReverse() {
  // Left Motor REV
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(ENA, HIGH);
  
  // Right Motor REV
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  digitalWrite(ENB, HIGH);
  
  Serial.println("Robot: REVERSE");
}

void robotStop() {
  // Disable both enable lines (coasts to stop)
  digitalWrite(ENA, LOW);
  digitalWrite(ENB, LOW);
  Serial.println("Robot: STOP");
}

void setup() {
  Serial.begin(115200);
  
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(ENA, OUTPUT);
  
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  pinMode(ENB, OUTPUT);
  
  robotStop(); // Safe start
  Serial.println("Dual Motor Controller ready.");
}

void loop() {
  robotForward();
  delay(3000);
  
  robotStop();
  delay(1000);
  
  robotReverse();
  delay(3000);
  
  robotStop();
  delay(1000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N Motor Driver**, and two **DC Motors** onto the canvas.
2. Wire ENA to **GPIO5**, ENB to **GPIO23**, IN1–IN4 to **GPIO18, 19, 21, 22**. Connect both motors to L298N outputs.
3. Paste the code and click **Run**.
4. Watch both motor widgets spin forward together, stop, reverse together, and stop.

## Expected Output
Serial Monitor:
```
Dual Motor Controller ready.
Robot: FORWARD
Robot: STOP
Robot: REVERSE
Robot: STOP
```

## Expected Canvas Behavior
* Both DC motor widgets spin clockwise for 3 seconds.
* Both motor widgets stop for 1 second.
* Both motor widgets spin counter-clockwise for 3 seconds.
* Both stop. Cycle repeats.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `IN1=HIGH, IN2=LOW` | Sets Left H-bridge to route forward current. |
| `IN3=HIGH, IN4=LOW` | Sets Right H-bridge to route forward current. |
| `digitalWrite(ENA, HIGH)` | Enables Left Motor drive. |
| `digitalWrite(ENB, HIGH)` | Enables Right Motor drive. |
| `digitalWrite(ENA, LOW)` | Disables both H-bridges, causing the robot to coast to a halt. |

## Hardware & Safety Concept: Dual H-Bridge Motor Control
Mobile robots utilize differential drive steering (two independent wheels). The L298N H-bridge module contains two separate driver circuits. Each driver has its own direction inputs and enable pin. By controlling the voltage polarity across each motor coil independently, the robot can be commanded to drive forward (both wheels forward), reverse (both wheels backward), or steer (wheels rotating in opposite directions). External batteries are mandatory because motors consume up to 800 mA during startup, which would trigger the ESP32's onboard thermal fuse if drawn from logic rails.

## Try This! (Challenges)
1. **Pivot steering**: Write a function `robotPivotLeft()` that drives the Left Motor in REVERSE and the Right Motor FORWARD to spin on the spot.
2. **Speed Scaling integration**: Use `ledcSetup` on pins 5 and 23 to run the robot at half speed (127 duty cycle).
3. **Collision alarm shutoff**: Connect a push button on GPIO 4 that acts as a bumper sensor, stopping both motors instantly when triggered.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| One motor spins backward | Terminals wired in reverse | Swap the two wire leads of the affected motor at the driver terminals |
| Motors hum but do not spin | Low battery voltage or missing GND link | Check external battery charge; verify L298N GND is wired to ESP32 GND |
| Only one motor operates | ENA or ENB jumper still connected | Ensure both black header jumpers are removed from the ENA/ENB pins on the driver |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [53 - ESP32 DC Motor Start/Stop](../intermediate/53-esp32-dc-motor-start-stop.md)
- [122 - ESP32 Mobile Robot Left/Right Steering](121-esp32-dual-motor-forward-reverse-drive.md#related-projects) (Next project)
- [123 - ESP32 Robot Speed Control](123-esp32-robot-speed-control.md)
