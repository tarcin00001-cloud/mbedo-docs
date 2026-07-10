# 45 - Suntracker Servo

Build a solar tracker that automatically rotates a servo toward the brightest light source.

## Goal
Learn how to read two independent analog light sensors (LDRs), compare their light levels, and rotate a servo motor incrementally to balance the sensor inputs.

## What You Will Build
Two LDR light sensors are mounted on opposite sides of a divider (simulating left and right). The Arduino reads both. If the left LDR sees more light, the servo rotates left. If the right LDR sees more light, the servo rotates right, eventually centering on the light source.

**Why A0, A1, and D9?** Pins A0 and A1 read the analog voltages from the left and right LDR sensors. Pin D9 drives the tracking servo motor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| LDR Light Sensor | `ldr` | Yes (2 required) | Yes (2 required) |
| Servo Motor | `servo` | Yes | Yes |
| 10k ohm resistor | `resistor` | Optional | Yes (2 required) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Left LDR Sensor | VCC | 5V | Power supply (5V) |
| Left LDR Sensor | AO | A0 | Analog signal (Left) |
| Left LDR Sensor | GND | GND | Ground reference |
| Right LDR Sensor | VCC | 5V | Power supply (5V) |
| Right LDR Sensor | AO | A1 | Analog signal (Right) |
| Right LDR Sensor | GND | GND | Ground reference |
| Servo Motor | VCC | 5V | Power (red wire) |
| Servo Motor | SIG | D9 | Control pin (orange wire) |
| Servo Motor | GND | GND | Ground reference (brown wire) |

## Code
```cpp
#include <Servo.h>

const int LDR_LEFT_PIN  = A0;
const int LDR_RIGHT_PIN = A1;
const int SERVO_PIN     = 9;

Servo trackerServo;

int servoAngle = 90; // Start at center position (90 degrees)
const int tolerance = 30; // Deadband to prevent jittering

void setup() {
  trackerServo.attach(SERVO_PIN);
  trackerServo.write(servoAngle);
  
  Serial.begin(9600);
  Serial.println("Suntracker Ready");
}

void loop() {
  int ldrLeft  = analogRead(LDR_LEFT_PIN);
  int ldrRight = analogRead(LDR_RIGHT_PIN);
  
  Serial.print("Left LDR: ");
  Serial.print(ldrLeft);
  Serial.print(" | Right LDR: ");
  Serial.println(ldrRight);

  // Calculate the difference between left and right light levels
  int diff = ldrLeft - ldrRight;
  
  // If left side is brighter (exceeding tolerance)
  if (diff > tolerance) {
    servoAngle = servoAngle - 5; // Move left
  } 
  // If right side is brighter (exceeding tolerance)
  else if (diff < -tolerance) {
    servoAngle = servoAngle + 5; // Move right
  }
  
  // Constrain servo limits between 0 and 180 degrees
  servoAngle = constrain(servoAngle, 0, 180);
  
  trackerServo.write(servoAngle);
  delay(50); // Speed of adjustment delay
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **two LDR Sensors**, and **Servo Motor** onto the canvas.
2. Connect Left LDR **VCC** → Arduino **5V**, **AO** → Arduino **A0**, and **GND** → Arduino **GND**.
3. Connect Right LDR **VCC** → Arduino **5V**, **AO** → Arduino **A1**, and **GND** → Arduino **GND**.
4. Connect Servo **VCC** → Arduino **5V**, **SIG** → Arduino **D9**, and **GND** → Arduino **GND**.
5. Paste the code into the editor.
6. Use the default Arduino interpreted runtime.
7. Click **Run**.
8. Open the **Terminal** tab in the bottom dock.
9. Double-click both LDRs, adjust their light level sliders independently, and watch the Servo turn to track the brighter sensor.

## Expected Output

Terminal:
```
Suntracker Ready
Left LDR: 500 | Right LDR: 500
Left LDR: 800 | Right LDR: 400
Left LDR: 720 | Right LDR: 450
...
```

### Expected Canvas Behavior

| Light level inputs | Difference calculation | Servo target Angle | Rotation Action |
| --- | --- | --- | --- |
| Equal (Left = Right) | Near 0 | No change | Locked on center |
| Left Brighter | > 30 | Decreases (e.g. -> 75) | Rotates Counter-clockwise |
| Right Brighter | < -30 | Increases (e.g. -> 105) | Rotates Clockwise |

The servo centers itself on whichever sensor has the higher light level slider position.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `int diff = ldrLeft - ldrRight;` | Calculates the imbalance between left and right light levels. |
| `constrain(servoAngle, 0, 180)` | Ensures the calculated rotation angle does not attempt to exceed the mechanical range limits of the servo. |
| `trackerServo.write(servoAngle)` | Sends the position pulse to update the motor. |

## Hardware & Safety Concept: Sensor Deadbands (Tolerance)
If the controller responded to even a 1-unit difference between the LDR readings, the servo would vibrate and jitter continuously because raw analog readings always have minor noise. Adding a **deadband tolerance** (e.g., 30) ensures the servo only rotates when there is a significant, real change in light direction, extending the service life of the motor.

## Try This! (Challenges)
1. **Finer Tracking**: Reduce the tolerance to `10` and step angle increment to `2` to make the tracking smoother and more accurate.
2. **Auto-Park**: Modify the code so that if both LDR sensors read below `100` (representing night), the servo automatically rotates to 90 degrees (facing straight up, ready for sunrise).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo rotates in the opposite direction | LDR sensor pins swapped | Swap the pin definitions in the code: `LDR_LEFT_PIN = A1` and `LDR_RIGHT_PIN = A0`. |
| Servo jitters constantly without stopping | Deadband tolerance too low | Increase the `tolerance` constant in the code to 50 or 60 to filter out noise. |

## Mode Notes
These patterns (dual analog sensor readings driving servo angular increments) are supported by MbedO interpreted mode.

## Related Projects
- [18 - Light Meter](../beginner/18-light-meter.md)
- [36 - Rain Servo Wiper](36-rain-servo-wiper.md)
- [46 - Potentiometer Servo](46-potentiometer-servo.md)
