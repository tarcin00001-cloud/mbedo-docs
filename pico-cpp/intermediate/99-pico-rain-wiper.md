# 99 - Pico Rain Wiper

Build an automated windshield wiper that sweeps a servo motor when a rain sensor detects moisture.

## Goal
Learn how to use digital input sensor triggers (rain sensor) to actuate dynamic loop movements on a servo motor.

## What You Will Build
An automatic windshield wiper system:
- **Rain Sensor DO (GP16)**: Reads `LOW` when wet.
- **Servo Motor (GP10)**: Sweeps back and forth continuously between 0° and 180° while rain is detected, and returns to 0° (parked position) when dry.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Rain Sensor Board | `button` | Yes (represented by switch button) | Yes |
| Servo Motor | `servo` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Rain Sensor | VCC | 3.3V | Power supply |
| Rain Sensor | DO | GP16 | Digital wet signal |
| Rain Sensor | GND | GND | Ground return |
| Servo Motor | Signal | GP10 | Servo sweep control |
| Servo Motor | Power | VBUS (5V) | Power supply |
| Servo Motor | Ground | GND | Ground return |

## Code
```cpp
#include <Servo.h>

const int RAIN_PIN  = 16;
const int SERVO_PIN = 10;

Servo wiperServo;

int angle = 0;
int stepDir = 1;

void setup() {
  pinMode(RAIN_PIN, INPUT);
  wiperServo.attach(SERVO_PIN);
  wiperServo.write(0); // Parked position
}

void loop() {
  int rainDetected = digitalRead(RAIN_PIN);

  // Active-LOW check: rain sensor pulls DO to GND when wet
  if (rainDetected == LOW) {
    // Increment angle step-by-step to avoid nested loops
    angle = angle + (stepDir * 10);
    
    if (angle >= 180) {
      angle = 180;
      stepDir = -1;
    }
    if (angle <= 0) {
      angle = 0;
      stepDir = 1;
    }
    
    wiperServo.write(angle);
    delay(30); // Control sweep speed
  } else {
    // If dry, return to parked position (0 degrees)
    if (angle > 0) {
      angle = angle - 5;
      if (angle < 0) { angle = 0; }
      wiperServo.write(angle);
      delay(20);
    }
  }

  delay(10);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Rain Sensor** (represented by switch button), and **Servo Motor** onto the canvas.
2. Connect Rain Sensor DO to **GP16**, Servo Signal to **GP10**, and connect power/ground.
3. Paste code, select the interpreted mode, and click **Run**.
4. Simulate rain by closing the switch contact representing the wet sensor on canvas, and watch the wiper servo sweep.

## Expected Output

Terminal:
```
Simulation active. Rain wiper control online.
```

## Expected Canvas Behavior
| Rain Sensor State | GP16 State | Servo Motor (GP10) Action |
| --- | --- | --- |
| Dry | HIGH | Parked at 0° |
| Wet (Raining) | LOW | Sweeping back and forth (0° ↔ 180°) |
| Dry again | HIGH | Returns smoothly to 0° |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `wiperServo.write(0)` | Parks the servo at 0 degrees when the sensor plate goes dry, preventing the blade from stopping mid-windshield. |

## Hardware & Safety Concept: Wiper Parking Logic
Windshield wiper controllers must ensure the blades return to a "parked" position out of the driver's view when turned off. In automotive engineering, this is managed by a mechanical park switch inside the gear assembly. In our software, the `else` block monitors the current angle and decrements it back to 0° step-by-step when rain stops, rather than halting instantly.

## Try This! (Challenges)
1. **Speed Control**: Adjust the step increment size based on rain intensity (e.g. sweep faster if an analog rain sensor value is very low).
2. **Indicator LED**: Turn ON an external blue LED on GP15 when the wiper is active.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Wiper stops instantly mid-sweep | Parking loop missing | Check that the `else` block containing the decremental park code is not blocked by a simple `write(0)` without delay loops. |

## Mode Notes
This multi-device control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [39 - Pico Rain Sensor](../../beginner/39-pico-rain-sensor.md)
- [51 - Pico Servo Sweep](../intermediate/51-pico-servo-sweep.md)
