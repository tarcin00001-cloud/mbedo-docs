# 115 - Servo Radar

Mount an HC-SR04 ultrasonic sensor on a servo and sweep it across four fixed angles, printing the distance measured at each bearing angle to the Serial Monitor — building the data for a simple radar display.

## Goal
Learn how to combine a servo sweep with distance measurements to acquire a multi-angle proximity scan, and understand how to structure polling-based radar data output without using for-loops.

## What You Will Build
The servo steps through four fixed bearing angles (0°, 45°, 90°, 135°) in sequence. At each position the sketch waits 250 ms for the servo to settle, fires the HC-SR04 ultrasonic pulse, and prints the angle and measured distance to the Serial Terminal:

```
Angle: 0   deg  Dist: 82 cm
Angle: 45  deg  Dist: 34 cm
Angle: 90  deg  Dist: 120 cm
Angle: 135 deg  Dist: 55 cm
```

After reaching 135° the servo returns to 0° and the scan repeats continuously, creating a sweep update roughly every second.

**Why D6, D8, D9?** D6 drives the servo PWM signal. D8 is the ultrasonic TRIG output. D9 is the ultrasonic ECHO input. These avoid SPI/I2C pins and the SoftwareSerial pins so the sketch can be easily combined with other modules.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `hc_sr04` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Servo Motor | Signal | D6 | PWM control |
| Servo Motor | VCC | 5V | Power supply |
| Servo Motor | GND | GND | Ground reference |
| HC-SR04 | VCC | 5V | Power supply |
| HC-SR04 | TRIG | D8 | Trigger pulse output |
| HC-SR04 | ECHO | D9 | Echo pulse input |
| HC-SR04 | GND | GND | Ground reference |

## Code
```cpp
#include <Servo.h>

const int SERVO_PIN = 6;
const int TRIG_PIN  = 8;
const int ECHO_PIN  = 9;

Servo radarServo;

int angle = 0;      // Current bearing angle
int step  = 0;      // 0=0deg 1=45deg 2=90deg 3=135deg

void setup() {
  Serial.begin(9600);

  radarServo.attach(SERVO_PIN);
  radarServo.write(0);

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  delay(500); // Allow servo to reach 0 degrees
  Serial.println("Servo Radar Active — scanning...");
}

void loop() {
  // Set angle based on step counter (no for-loop, no array)
  if (step == 0) {
    angle = 0;
  } else if (step == 1) {
    angle = 45;
  } else if (step == 2) {
    angle = 90;
  } else {
    angle = 135;
  }

  // Move servo to bearing
  radarServo.write(angle);
  delay(250); // Wait for servo to settle

  // Fire HC-SR04 pulse
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH);
  long dist = duration * 0.034 / 2;

  // Print radar data point
  Serial.print("Angle: ");
  if (angle < 100) Serial.print(" "); // Align columns
  if (angle < 10)  Serial.print(" ");
  Serial.print(angle);
  Serial.print(" deg  Dist: ");
  Serial.print(dist);
  Serial.println(" cm");

  // Advance to next step (cycle 0 -> 1 -> 2 -> 3 -> 0)
  step = step + 1;
  if (step > 3) {
    step = 0;
    Serial.println("--- sweep complete ---");
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **Servo Motor**, and **HC-SR04** onto the canvas.
2. Connect Servo: **Signal** → **D6**, **VCC** → **5V**, **GND** → **GND**.
3. Connect HC-SR04: **TRIG** → **D8**, **ECHO** → **D9**, **VCC** → **5V**, **GND** → **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Watch the servo step through four angles on the canvas and observe the distance readings at each bearing in the Terminal.
8. Adjust the HC-SR04 distance slider on the canvas between servo steps to simulate objects at different bearings.

## Expected Output

Terminal:
```
Servo Radar Active — scanning...
Angle:   0 deg  Dist: 82 cm
Angle:  45 deg  Dist: 34 cm
Angle:  90 deg  Dist: 120 cm
Angle: 135 deg  Dist: 55 cm
--- sweep complete ---
Angle:   0 deg  Dist: 80 cm
...
```

## Expected Canvas Behavior
| Step | Servo Angle | HC-SR04 Reading | Terminal Output |
| --- | --- | --- | --- |
| 0 | 0 deg | 82 cm | `Angle:   0 deg  Dist: 82 cm` |
| 1 | 45 deg | 34 cm | `Angle:  45 deg  Dist: 34 cm` |
| 2 | 90 deg | 120 cm | `Angle:  90 deg  Dist: 120 cm` |
| 3 | 135 deg | 55 cm | `Angle: 135 deg  Dist: 55 cm` |
| — | Returns to 0 | — | `--- sweep complete ---` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `int step = 0` global | Tracks which of the four bearing angles the radar is currently pointing at across loop iterations. |
| `if (step == 0) angle = 0; else if …` | Translates the step counter into a servo angle without using an array — safe for MbedO interpreted mode. |
| `radarServo.write(angle)` then `delay(250)` | Commands the servo to move then waits for it to physically settle before firing the ultrasonic pulse. |
| `duration * 0.034 / 2` | Converts echo pulse microseconds to centimeters using the speed of sound (340 m/s). |
| `step = step + 1; if (step > 3) step = 0` | Manually wraps the step counter back to 0 after the last angle — equivalent to `step % 4` without the modulo operator. |

## Hardware & Safety Concept: Pulse-Echo Ultrasonic Ranging
The HC-SR04 emits a 40 kHz ultrasonic burst from its transmitter and listens for the echo on its receiver. The time between TRIG HIGH and ECHO HIGH falling edge is proportional to the round-trip distance (distance = duration × speed_of_sound / 2). The sensor's effective range is approximately 2–400 cm. Objects outside this range or at very sharp angles may not return a measurable echo, giving a reading of 0.

## Try This! (Challenges)
1. **Eight-Point Sweep**: Add four more `else if` branches to step through 0°, 22.5°, 45°, 67.5°, 90°, 112.5°, 135°, 157.5° (eight bearings per sweep) for higher angular resolution.
2. **Serial Plotter Radar**: Format the output as `angle,distance` comma-separated values to feed directly into the Arduino IDE Serial Plotter for a visual polar display.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo stays at 0 degrees | `attach()` not called or wrong pin | Confirm `radarServo.attach(SERVO_PIN)` is called in `setup()` and SERVO_PIN = 6. |
| Distance always reads 0 | TRIG/ECHO pins swapped | TRIG must be OUTPUT on D8 and ECHO must be INPUT on D9. |
| All angles measure the same distance | Servo not moving before measurement | Increase `delay(250)` after `radarServo.write()` to allow more settling time. |

## Mode Notes
`Servo.write()`, `pulseIn()`, `digitalWrite()`, and `delayMicroseconds()` are all supported by MbedO interpreted mode.

## Related Projects
- [58 - Servo Radar](../intermediate/58-servo-radar.md)
- [54 - Distance Serial](../intermediate/54-distance-serial.md)
- [45 - Suntracker Servo](../intermediate/45-suntracker-servo.md)
- [108 - Digital Spirit Level](108-digital-spirit-level.md)
