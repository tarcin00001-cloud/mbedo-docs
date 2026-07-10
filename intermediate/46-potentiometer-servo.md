# 46 - Potentiometer Servo

Control a servo motor's angle directly using a potentiometer dial.

## Goal
Learn how to read an analog voltage from a dial (0 to 5V), map it to an angular scale (0 to 180 degrees), and position a servo motor dynamically.

## What You Will Build
As you turn the potentiometer dial on the canvas, the servo motor connected to D9 rotates immediately to match the angle of the dial.

**Why A0 and D9?** Pin A0 converts the dial position voltage. Pin D9 outputs the control pulses required to position the servo shaft.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Potentiometer | 1 | 5V | Positive voltage reference |
| Potentiometer | 2 (Wiper) | A0 | Analog signal connection |
| Potentiometer | 3 | GND | Ground reference |
| Servo Motor | VCC | 5V | Power (red wire) |
| Servo Motor | SIG | D9 | Control pin (orange wire) |
| Servo Motor | GND | GND | Ground reference (brown wire) |

## Code
```cpp
#include <Servo.h>

const int POT_PIN   = A0;
const int SERVO_PIN = 9;

Servo myServo;

void setup() {
  myServo.attach(SERVO_PIN);
  Serial.begin(9600);
  Serial.println("Potentiometer Servo Controller Ready");
}

void loop() {
  int rawVal = analogRead(POT_PIN);
  
  // Map 0-1023 input range directly to 0-180 degrees
  int angle = map(rawVal, 0, 1023, 0, 180);
  
  myServo.write(angle);
  
  Serial.print("Pot Raw: ");
  Serial.print(rawVal);
  Serial.print(" -> Servo Angle: ");
  Serial.println(angle);
  
  delay(50); // Refresh 20 times per second
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **Potentiometer**, and **Servo Motor** onto the canvas.
2. Connect Potentiometer **pin 1** to Arduino **5V**, **pin 2 (wiper)** to Arduino **A0**, and **pin 3** to Arduino **GND**.
3. Connect Servo **VCC** to Arduino **5V**, **SIG** to Arduino **D9**, and **GND** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the Potentiometer, adjust its knob value, and watch the Servo turn.

## Expected Output

Terminal:
```
Potentiometer Servo Controller Ready
Pot Raw: 0 -> Servo Angle: 0
Pot Raw: 512 -> Servo Angle: 90
Pot Raw: 1023 -> Servo Angle: 180
...
```

### Expected Canvas Behavior

| Potentiometer Position | Pin A0 Reading | Servo angle target | Servo position |
| --- | --- | --- | --- |
| 0% (Left) | 0 | 0 degrees | Parked Left |
| 50% (Center) | 512 | 90 degrees | Centered |
| 100% (Right) | 1023 | 180 degrees | Parked Right |

The servo angle updates instantly as the potentiometer knob is turned.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `int angle = map(rawVal, 0, 1023, 0, 180)` | Scales the 10-bit analog range (0-1023) directly to the servo's mechanical limit range of 180 degrees. |
| `myServo.write(angle)` | Sends a pulses train corresponding to the target angle, directing the servo controller to position the shaft. |

## Hardware & Safety Concept: PWM Servo Signals
Servo motors do not run on pure analog voltage. Instead, they use a specific control signal called **Pulse Width Modulation (PWM)** or **Pulse Position Modulation (PPM)**.
- The control signal is a repeating 50 Hz square wave.
- The duration of the positive pulse (width) tells the servo where to rotate.
  - A 1.0 ms pulse corresponds to 0 degrees.
  - A 1.5 ms pulse corresponds to 90 degrees.
  - A 2.0 ms pulse corresponds to 180 degrees.
The Arduino `Servo` library handles this timing behind the scenes.

## Try This! (Challenges)
1. **Narrow Sweeps**: Modify the `map()` arguments so that the servo only sweeps between 45 and 135 degrees.
2. **Reverse Mapping**: Reverse the mapping so that turning the dial clockwise rotates the servo counter-clockwise (i.e. map 0-1023 to 180-0).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo does not move or hums | Wrong pin used | Ensure the Servo SIG wire is connected to D9, matching `SERVO_PIN = 9`. |
| Servo moves to max angle and vibrates | Potentiometer outer pins shorted | Ensure Pin 1 is connected to 5V and Pin 3 to GND. |

## Mode Notes
These patterns (analog reads, `map()` math, and servo library writes) are supported by MbedO interpreted mode.

## Related Projects
- [14 - Dial LED Brightness](../beginner/14-dial-led-brightness.md)
- [36 - Rain Servo Wiper](36-rain-servo-wiper.md)
- [45 - Suntracker Servo](45-suntracker-servo.md)
