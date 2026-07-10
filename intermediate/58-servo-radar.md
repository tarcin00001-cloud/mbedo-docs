# 58 - Servo Radar

Build a radar scanner that sweeps a servo motor and prints obstacle distances at each angle.

## Goal
Learn how to coordinate a servo motor sweep and ultrasonic distance measurements concurrently without using blocking loops, outputting mapped coordinate data.

## What You Will Build
The servo motor connected to D9 sweeps back and forth between 0 and 180 degrees. At each 15-degree step, the Arduino pauses briefly, measures the distance to the nearest obstacle using the HC-SR04, and prints the current angle and distance to the Terminal.

**Why D2, D3, and D9?** Pins D2/D3 manage the distance checks. Pin D9 sweeps the radar mount.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-SR04 Ultrasonic | `hc_sr04` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-SR04 Sensor | VCC | 5V | Power supply (5V) |
| HC-SR04 Sensor | TRIG | D2 | Trigger connection |
| HC-SR04 Sensor | ECHO | D3 | Echo connection |
| HC-SR04 Sensor | GND | GND | Ground reference |
| Servo Motor | VCC | 5V | Power (red wire) |
| Servo Motor | SIG | D9 | Control pin (orange wire) |
| Servo Motor | GND | GND | Ground reference (brown wire) |

## Code
```cpp
#include <Servo.h>

const int TRIG_PIN  = 2;
const int ECHO_PIN  = 3;
const int SERVO_PIN = 9;

Servo radarServo;

int angle = 0;       // Current sweep angle of the radar
int sweepDir = 1;    // Direction: 1 = clockwise, -1 = counter-clockwise

void setup() {
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  radarServo.attach(SERVO_PIN);
  radarServo.write(angle); // Initialize at 0 degrees
  
  Serial.begin(9600);
  Serial.println("Radar Scanner Ready");
}

void loop() {
  // 1. Move servo to the next step
  angle = angle + (sweepDir * 15);
  
  // Reverse direction at the boundaries
  if (angle >= 180) {
    angle = 180;
    sweepDir = -1;
  }
  if (angle <= 0) {
    angle = 0;
    sweepDir = 1;
  }
  
  radarServo.write(angle);
  
  // 2. Short delay to allow the servo motor to reach the target angle
  delay(150); 
  
  // 3. Trigger ultrasonic measurement
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  long duration = pulseIn(ECHO_PIN, HIGH);
  long distance = duration * 0.034 / 2;
  
  // 4. Print mapped angle and distance
  Serial.print("Angle: ");
  Serial.print(angle);
  Serial.print(" | Distance: ");
  Serial.print(distance);
  Serial.println(" cm");
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **HC-SR04 Sensor**, and **Servo Motor** onto the canvas.
2. Connect HC-SR04 **VCC** to Arduino **5V**, **TRIG** to Arduino **D2**, **ECHO** to Arduino **D3**, and **GND** to Arduino **GND**.
3. Connect Servo **VCC** → Arduino **5V**, **SIG** → Arduino **D9**, and **GND** → Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the HC-SR04, adjust the distance slider, and watch the values print alongside their respective angles in the Terminal.

## Expected Output

Terminal:
```
Radar Scanner Ready
Angle: 15 | Distance: 120 cm
Angle: 30 | Distance: 120 cm
Angle: 45 | Distance: 80 cm
...
```

### Expected Canvas Behavior

| Loop Step | Servo Angle state | Distance Read action | Servo mechanical action |
| --- | --- | --- | --- |
| 1 | 15 degrees | Measures A0/A1 | Turn clockwise |
| ... | ... | ... | ... |
| 12 | 180 degrees | Measures A0/A1 | Hits limit -> reverses |
| 13 | 165 degrees | Measures A0/A1 | Turn counter-clockwise |

The servo sweeps continuously on the canvas while distance logs update in the Terminal.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `angle = angle + (sweepDir * 15);` | Increments or decrements the angle by 15 degrees depending on the current direction, achieving a continuous sweep without blocking `for` loops. |
| `delay(150)` | Gives the physical servo motor gears enough time to rotate to the target angle before launching the ultrasonic sound burst. |

## Hardware & Safety Concept: Sonar Scanning
Radar and sonar scanners sweep a sensor beam through an arc to construct a 2D map of the surrounding environment. In robotic applications, this coordinate map (angle + distance) is used for obstacle avoidance, wall-following algorithms, and building room layouts (SLAM).

## Try This! (Challenges)
1. **Finer Resolution**: Modify the code to sweep in steps of `5` degrees instead of `15`. Adjust the delay to `80` ms.
2. **Alarm Zone Indicator**: Wire an LED to D13 and make it light up if the scanner detects an object closer than `30` cm at *any* angle during the sweep.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Distance prints as 0 at all angles | Trigger or Echo wire loose | Verify the HC-SR04 connections. Check that the echo pin matches D3. |
| Sweep does not change direction | Comparison boundaries wrong | Ensure you check `angle >= 180` and `angle <= 0` exactly in your logic checks. |

## Mode Notes
The `Servo` library is shimmed. The sweep logic uses step-wise variable modification rather than a `for` loop to comply with the interpreted mode constraints.

## Related Projects
- [36 - Rain Servo Wiper](36-rain-servo-wiper.md)
- [54 - Distance Serial](54-distance-serial.md)
- [56 - Parking Beeper](56-parking-beeper.md)
