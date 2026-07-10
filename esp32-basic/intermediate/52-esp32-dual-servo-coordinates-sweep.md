# 52 - ESP32 Dual Servo Coordinates Sweep

Control two servo motors simultaneously on independent LEDC channels, sweeping them in coordinated opposing directions to simulate a pan-tilt or scissor mechanism.

## Goal
Learn how to attach two independent servo objects to two different GPIO pins and synchronise their movement with a single loop — each servo moving in the opposite direction to its partner.

## What You Will Build
Servo A on GPIO 5 sweeps 0°→180°; Servo B on GPIO 18 sweeps 180°→0° simultaneously, creating an opposing cross-sweep. At the ends they pause and reverse together.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Servo Motor A (SG90) | `servo` | Yes | Yes |
| Servo Motor B (SG90) | `servo` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Servo A | Signal | GPIO5 | Orange | Servo A PWM control |
| Servo A | VCC | 5V (Vin) | Red | Servo A power |
| Servo A | GND | GND | Black | Servo A ground |
| Servo B | Signal | GPIO18 | Yellow | Servo B PWM control |
| Servo B | VCC | 5V (Vin) | Red | Servo B power — share 5 V rail |
| Servo B | GND | GND | Black | Servo B ground — share GND rail |

> **Wiring tip:** Both servos share the same 5 V and GND rails. Two SG90 servos can stall at 1 A combined — use a USB power supply rated at ≥1.5 A or add an external 5 V regulator. Do not connect both servo VCC wires to the ESP32's onboard 3V3 regulator.

## Code
```cpp
// Dual Servo Opposing Sweep
#include <ESP32Servo.h>

const int SERVO_A_PIN = 5;
const int SERVO_B_PIN = 18;

Servo servoA;
Servo servoB;

void setup() {
  servoA.attach(SERVO_A_PIN);
  servoB.attach(SERVO_B_PIN);
  Serial.begin(115200);
  Serial.println("Dual Servo Sweep ready.");
}

void loop() {
  // Forward: A goes 0→180, B goes 180→0 simultaneously
  Serial.println("Forward sweep");
  for (int a = 0; a <= 180; a++) {
    servoA.write(a);
    servoB.write(180 - a);   // B is always the mirror of A
    delay(10);
  }
  delay(500);

  // Reverse: A goes 180→0, B goes 0→180
  Serial.println("Reverse sweep");
  for (int a = 180; a >= 0; a--) {
    servoA.write(a);
    servoB.write(180 - a);
    delay(10);
  }
  delay(500);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and two **Servo** components onto the canvas.
2. Connect Servo A **signal** to **GPIO5**, Servo B **signal** to **GPIO18**.
3. Paste the code and click **Run**.
4. Observe the two servo widgets moving in opposite directions in synchrony.

## Expected Output
Serial Monitor:
```
Dual Servo Sweep ready.
Forward sweep
Reverse sweep
Forward sweep
```

## Expected Canvas Behavior
* Servo A widget arm moves left to right while Servo B arm moves right to left simultaneously.
* Both pause at the extremes before reversing together.
* Motion is smooth and continuous.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `servoA.attach(SERVO_A_PIN)` | Allocates an LEDC channel for Servo A on GPIO 5. |
| `servoB.attach(SERVO_B_PIN)` | Allocates a second LEDC channel for Servo B on GPIO 18. |
| `servoB.write(180 - a)` | The expression `180 - a` is the mirror of `a` — when A is at 0°, B is at 180°, and vice versa. |
| `delay(10)` | 10 ms per step commands both servos at the same rate, keeping them synchronised. |

## Hardware & Safety Concept: Independent LEDC Channels for Multi-Servo Control
The ESP32 has 16 LEDC channels. The ESP32Servo library automatically assigns one channel per `Servo.attach()` call. Each channel operates independently with its own timer, frequency, and pulse width — so two (or more) servos can be commanded to different positions simultaneously without any interference. This is the basis of **multi-joint robotic systems**: a 6-DOF robotic arm uses six independent servo channels, each updated at 50 Hz. The critical constraint is power: each additional servo adds up to 500 mA stall current to the supply rail. Always size your power supply to handle the sum of all servo stall currents.

## Try This! (Challenges)
1. **Scissor mode**: Make both servos move in the same direction (remove the `180 - a` inversion) to simulate a scissor lift.
2. **Independent speed**: Give each servo a different `delay` per step so they reach their endpoints at different times.
3. **Joystick control**: Map the joystick X-axis (GPIO 34) to Servo A and Y-axis (GPIO 35) to Servo B for a manual pan-tilt system.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Only one servo moves | Second `attach()` failed | Ensure GPIO 18 is not used by another peripheral |
| Servos move but jerk | Power supply sag | Add a 100 µF capacitor across the shared 5 V and GND rail |
| Both servos start at 90° not 0° | No initial position set | Add `servoA.write(0); servoB.write(180); delay(500);` in `setup()` |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [51 - ESP32 Servo Motor 180° Sweep](51-esp32-servo-motor-180-sweep.md)
- [46 - ESP32 Potentiometer Servo Control](../beginner/46-esp32-potentiometer-servo-control.md)
- [97 - ESP32 Joystick Dual Axis Servo](97-esp32-joystick-dual-axis-servo.md)
