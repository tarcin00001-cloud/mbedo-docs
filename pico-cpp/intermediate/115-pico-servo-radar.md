# 115 - Pico Servo Radar

Build a physical radar scanner that sweeps an ultrasonic sensor on a servo motor and prints polar coordinate logs.

## Goal
Learn how to sweep a servo motor incrementally (no loops) while reading distances and printing matching polar log streams.

## What You Will Build
An ultrasonic radar tracker:
- **Servo Motor (GP10)**: Sweeps an ultrasonic sensor incrementally back and forth.
- **HC-SR04 Sensor (Trig GP14, Echo GP15)**: Measures distance at each step.
- **Serial Monitor**: Logs angle and distance (e.g. `120,45` representing 120° and 45 cm) to map surroundings.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| Resistors | `resistor` | Optional | Yes (voltage divider) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Servo Motor | Signal | GP10 | Sweep drive control |
| Servo Motor | Power | VBUS (5V) | Power supply |
| Servo Motor | Ground | GND | Ground return |
| HC-SR04 | VCC | 5V | Sensor power |
| HC-SR04 | Trig | GP14 | Trigger signal |
| HC-SR04 | Echo | GP15 | Echo signal (requires 5V to 3.3V divider) |
| HC-SR04 | GND | GND | Ground reference |

## Code
```cpp
#include <Servo.h>

const int SERVO_PIN = 10;
const int TRIG_PIN  = 14;
const int ECHO_PIN  = 15;

Servo radarServo;

int angle = 0;
int sweepDir = 5; // Sweep increment step

void setup() {
  Serial.begin(9600);
  radarServo.attach(SERVO_PIN);
  radarServo.write(angle);

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  digitalWrite(TRIG_PIN, LOW);
  delay(1000);
}

void loop() {
  // Move servo to the current angle step
  radarServo.write(angle);
  delay(150); // Allow servo time to reach the target angle

  // Trigger ultrasonic measurement
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH);
  int distance = duration * 0.0343 / 2;

  // Print polar coordinates: Angle,Distance
  Serial.print(angle);
  Serial.print(",");
  Serial.println(distance);

  // Increment angle for the sweep
  angle = angle + sweepDir;

  // Reverse sweep direction at limit boundaries (0 to 180 degrees)
  if (angle >= 180) {
    angle = 180;
    sweepDir = -5;
  }
  if (angle <= 0) {
    angle = 0;
    sweepDir = 5;
  }
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Servo**, and **HC-SR04 Sensor** onto the canvas.
2. Connect Servo to **GP10**, HC-SR04 Trig to **GP14**, and Echo to **GP15**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Observe the servo sweep back and forth while coordinate streams print to the Serial console.

## Expected Output

Terminal:
```
0,120
5,120
10,115
15,45
```

## Expected Canvas Behavior
* Startup: Servo rotates to 0°.
* Running: Servo sweeps incrementally in 5° steps from 0° to 180°, and then back to 0° continuously.
* Serial Monitor: Logs the polar coordinates at each step.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Serial.print(angle); Serial.print(","); Serial.println(distance);` | Prints data in a standard comma-separated values format, allowing visualization software to map surroundings. |

## Hardware & Safety Concept: Radar Sweep Stabilization
When mounting sensors on moving servo horns, rapid accelerations and stops generate mechanical vibration and shaking. This vibration introduces noise into ultrasonic distance readings, as sound waves bounce off at varying angles. Adding a short settle delay (e.g. `delay(150)`) between servo movements ensures the sensor is stable before triggering a measurement.

## Try This! (Challenges)
1. **Critical Target Alarm**: Sound a buzzer on GP11 if an object is detected closer than 20 cm at any sweep angle.
2. **Visual Target Finder**: Turn ON a red LED on GP13 only when a target is detected within 30 cm, and turn it OFF if the path is clear.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Distance always reads `0` at certain angles | Echo bounce scattering | Ultrasonic sensors require perpendicular surfaces to reflect sound waves back. Angle offsets > 15 degrees can scatter the signal. |

## Mode Notes
This multi-device control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [46 - Pico Pot Servo](../../beginner/46-pico-pot-servo.md)
- [75 - Pico Ultrasonic Serial](../intermediate/75-pico-ultrasonic-serial.md)
- [114 - Pico Obstacle Avoider](114-pico-obstacle-avoider.md)
