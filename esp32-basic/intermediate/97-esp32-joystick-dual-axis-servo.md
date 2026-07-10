# 97 - ESP32 Joystick Dual Axis Servo

Build a pan-and-tilt servo controller that uses a dual-axis analog joystick to position two independent servo motors in real time.

## Goal
Learn how to read multiple analog inputs, apply dead-zones to filter center drift, and coordinate multi-axis servo PWM control loops.

## What You Will Build
A joystick module's VRx is connected to GPIO 34, and VRy to GPIO 35. Two SG90 servos are connected to GPIO 13 (Pan) and GPIO 14 (Tilt). The joystick displacement is mapped directly to the servo angles.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Analog Dual-Axis Joystick Module | `joystick` | Yes | Yes |
| Servo Motor A (SG90 - Pan) | `servo` | Yes | Yes |
| Servo Motor B (SG90 - Tilt) | `servo` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Joystick | VCC | 3V3 | Red | Power |
| Joystick | GND | GND | Black | Ground reference |
| Joystick | VRx | GPIO34 | Yellow | Horizontal axis |
| Joystick | VRy | GPIO35 | Green | Vertical axis |
| Servo A (Pan) | Signal | GPIO13 | Orange | Pan PWM control |
| Servo A (Pan) | VCC / GND | 5V / GND | Red / Black | Power rails |
| Servo B (Tilt) | Signal | GPIO14 | Yellow | Tilt PWM control |
| Servo B (Tilt) | VCC / GND | 5V / GND | Red / Black | Power rails |

> **Wiring tip:** Connect both servo VCC lines to the ESP32 5V (Vin) pin. The joystick VRx and VRy output lines connect to the input-only pins GPIO 34 and 35.

## Code
```cpp
// Joystick Dual Axis Servo (Pan & Tilt)
#include <ESP32Servo.h>

const int JOY_X = 34;
const int JOY_Y = 35;

const int PAN_PIN = 13;
const int TILT_PIN = 14;

Servo panServo;
Servo tiltServo;

// Center dead-zone range for joystick
const int CENTER_MIN = 1900;
const int CENTER_MAX = 2200;

void setup() {
  Serial.begin(115200);
  
  panServo.attach(PAN_PIN);
  tiltServo.attach(TILT_PIN);
  
  // Set initial default positions (90 degrees = centered)
  panServo.write(90);
  tiltServo.write(90);
  
  Serial.println("Pan-Tilt Controller ready.");
}

void loop() {
  int rawX = analogRead(JOY_X);
  int rawY = analogRead(JOY_Y);
  
  int panAngle = 90;
  int tiltAngle = 90;
  
  // Calculate Pan Angle (X-Axis)
  // If joystick is in the dead-zone, keep centered at 90°
  if (rawX >= CENTER_MIN && rawX <= CENTER_MAX) {
    panAngle = 90;
  } else {
    panAngle = map(rawX, 0, 4095, 0, 180);
  }
  
  // Calculate Tilt Angle (Y-Axis)
  if (rawY >= CENTER_MIN && rawY <= CENTER_MAX) {
    tiltAngle = 90;
  } else {
    tiltAngle = map(rawY, 0, 4095, 0, 180);
  }
  
  // Constrain bounds to prevent hitting endstops
  panAngle = constrain(panAngle, 10, 170);
  tiltAngle = constrain(tiltAngle, 10, 170);
  
  // Actuate servos
  panServo.write(panAngle);
  tiltServo.write(tiltAngle);
  
  Serial.print("Joy X: "); Serial.print(rawX);
  Serial.print(" -> Pan: "); Serial.print(panAngle);
  Serial.print(" | Joy Y: "); Serial.print(rawY);
  Serial.print(" -> Tilt: "); Serial.println(tiltAngle);
  
  delay(30); // 30Hz update frequency for smooth movements
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Joystick**, and two **Servos** onto the canvas.
2. Wire the X-axis to **GPIO34**, Y-axis to **GPIO35**, Pan servo to **GPIO13**, and Tilt servo to **GPIO14**.
3. Paste the code and click **Run**.
4. Drag the joystick widget in any direction. Watch the two servos follow the movement coordinates.

## Expected Output
Serial Monitor:
```
Pan-Tilt Controller ready.
Joy X: 2048 -> Pan: 90 | Joy Y: 2048 -> Tilt: 90
Joy X: 4095 -> Pan: 170 | Joy Y: 1024 -> Tilt: 45
Joy X: 0 -> Pan: 10 | Joy Y: 3980 -> Tilt: 170
```

## Expected Canvas Behavior
* Moving the joystick widget left and right rotates the Pan servo.
* Moving the joystick widget up and down rotates the Tilt servo.
* When the joystick is released, both servos return to their 90-degree center positions.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `rawX >= CENTER_MIN && rawX <= CENTER_MAX` | Ignores joystick center drift to prevent servo jitter when idle. |
| `constrain(..., 10, 170)` | Protects servo gear mechanisms from binding at extreme 0° or 180° limits. |
| `panServo.write(panAngle)` | Outputs the PWM pulses to pan the servo. |

## Hardware & Safety Concept: Pan-Tilt Power Isolation
Pan-tilt assemblies are often used to position cameras or sensors. Because two servos move at the same time, the sudden current draw can exceed 1A. In hardware layouts, power the servos from an external 5V regulator, and ensure only the control signal wires are connected to the ESP32 GPIOs (with a shared GND).

## Try This! (Challenges)
1. **Incremental Positional Jogging**: Modify the code so that moving the joystick shifts the current angle incrementally (like a mouse pointer) instead of absolute mapping.
2. **Servo Speed Controller**: Add a potentiometer (GPIO 36) that adjusts the transition speed of the servos.
3. **Joystick Click Center Reset**: Wire the joystick's SW push button to GPIO 4. If clicked, return the servos to 90 degrees instantly.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servos jitter constantly | Unfiltered center drift or electrical noise | Widen the `CENTER_MIN` and `CENTER_MAX` bounds; add a 100 nF capacitor |
| ESP32 resets when joystick moves | High current draw from 3V3 rail | Move servo VCC wires to the 5V (Vin) pin |
| Servos move in opposite directions | Axis mapping inverted | Invert the map function constraints (e.g. `map(raw, 0, 4095, 180, 0)`) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [46 - ESP32 Potentiometer Servo Control](../beginner/46-esp32-potentiometer-servo-control.md)
- [47 - ESP32 Analog Joystick X-Axis Logging](../beginner/47-esp32-analog-joystick-x-axis-logging.md)
- [52 - ESP32 Dual Servo Coordinates Sweep](52-esp32-dual-servo-coordinates-sweep.md)
