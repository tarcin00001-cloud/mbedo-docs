# 36 - Rain Servo Wiper

Build an automated windshield wiper that sweeps a servo motor when rain is detected.

## Goal
Learn how to sweep a servo motor continuously based on analog sensor thresholds using loop-free increment variables that are compatible with the interpreted runtime.

## What You Will Build
When the rain sensor detects wetness below 700 (remember: lower is wetter), a servo motor connected to D9 sweeps back and forth from 0 to 180 degrees to simulate a windshield wiper. When dry, the servo returns to 0 degrees and parks.

**Why A0 and D9?** Pin A0 measures the analog wetness level. Pin D9 connects to the servo signal wire to generate the PWM pulse train needed for angular positioning.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Rain Sensor | `rain_sensor` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Rain Sensor | VCC | 5V | Power supply (5V) |
| Rain Sensor | AO (Analog Out) | A0 | Analog signal connection |
| Rain Sensor | GND | GND | Ground reference |
| Servo Motor | VCC | 5V | Power (red wire) |
| Servo Motor | SIG (Signal) | D9 | Control pin (orange wire) |
| Servo Motor | GND | GND | Ground reference (brown wire) |

## Code
```cpp
#include <Servo.h>

const int RAIN_PIN = A0;
const int SERVO_PIN = 9;

Servo wiperServo;

int angle = 0;       // Current angle of the servo wiper
int sweepDir = 1;    // Direction: 1 = sweeping up, -1 = sweeping down

void setup() {
  wiperServo.attach(SERVO_PIN);
  wiperServo.write(0); // Park the wiper at 0 degrees
  
  Serial.begin(9600);
  Serial.println("Rain Wiper Controller Ready");
}

void loop() {
  int rainVal = analogRead(RAIN_PIN);
  
  Serial.print("Rain wetness: ");
  Serial.println(rainVal);

  // If rain is detected (threshold below 700)
  if (rainVal < 700) {
    // Increment the angle by 15 degrees in the current direction
    angle = angle + (sweepDir * 15);
    
    // Check limits and reverse direction
    if (angle >= 180) {
      angle = 180;
      sweepDir = -1; // Sweep back down
    }
    if (angle <= 0) {
      angle = 0;
      sweepDir = 1;  // Sweep back up
    }
    
    wiperServo.write(angle);
    delay(40); // Controls the speed of the sweep
  } else {
    // If dry, return to park position
    angle = 0;
    wiperServo.write(angle);
    delay(200);
  }
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **Rain Sensor**, and **Servo Motor** onto the canvas.
2. Connect Rain Sensor **VCC** to Arduino **5V**, **AO** to Arduino **A0**, and **GND** to Arduino **GND**.
3. Connect Servo **VCC** to Arduino **5V**, **SIG** to Arduino **D9**, and **GND** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the Rain Sensor, adjust the "wetness" slider, and watch the Servo sweep back and forth.

## Expected Output

Terminal:
```
Rain Wiper Controller Ready
Rain wetness: 1023
Rain wetness: 420
Rain wetness: 420
...
```

### Expected Canvas Behavior

| Moisture State | Pin A0 Reading | Servo State | Sweep Action |
| --- | --- | --- | --- |
| Dry | > 700 | Parked at 0 degrees | Stopped |
| Wet (Rain) | < 700 | Moving (0 to 180 degrees) | Continuous back-and-forth sweep |

The servo sweeps continuously as long as the rain sensor wetness is active.

## Code Walkthrough

| Variable / Statement | What It Does |
| --- | --- |
| `wiperServo.attach(SERVO_PIN)` | Attaches the Servo object to pin D9. |
| `angle = angle + (sweepDir * 15);` | Increments or decrements the position variable. By running this step-by-step each loop iteration, we achieve a sweep without using a blocking `for` loop. |
| `wiperServo.write(angle)` | Sends the angular command pulse to the servo. |

## Hardware & Safety Concept: Servo Power Draw
Servo motors contain DC motors and gearboxes. When sweeping or under load, a standard hobby servo (like SG90) can draw currents exceeding **500 mA**. 
- The Arduino Uno 5V pin can safely supply around 400-500 mA when powered via USB.
- Connecting multiple servos or high-power servos directly to the Arduino's 5V pin can cause the Arduino to brownout and reset. For real projects, always power your servos from an external 5V power supply, ensuring you connect the external ground to the Arduino ground.

## Try This! (Challenges)
1. **Speed Adjuster**: Change the delay from `40` to `20` to speed up the wiper sweeps, or `100` to slow it down.
2. **Double Sweep**: Make the servo sweep between 30 and 150 degrees instead of the full range.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo does not move when wet | Signal wire on wrong pin | Make sure the Servo signal pin is wired to D9, matching `SERVO_PIN = 9`. |
| Servo moves erratically or Arduino resets | Current draw too high | In physical circuits, power the servo using a separate 5V regulator. |

## Mode Notes
The `Servo` library is shimmed by the interpreter. The sweep logic uses step-wise variable modification rather than a `for` loop to comply with the interpreted mode constraints.

## Related Projects
- [25 - Rain Level Print](../beginner/25-rain-level-print.md)
- [46 - Potentiometer Servo](46-potentiometer-servo.md)
