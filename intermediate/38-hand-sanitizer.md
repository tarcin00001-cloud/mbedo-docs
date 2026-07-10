# 38 - Hand Sanitizer Dispenser

Build a touch-free automatic dispenser using an IR sensor and a servo motor.

## Goal
Learn how to use a proximity sensor (IR Sensor) to trigger a precise mechanical movement (Servo motor) for dispensing liquid, utilizing lockout delays to prevent double-triggering.

## What You Will Build
When a hand is placed in front of the IR proximity sensor, the digital output drops to `LOW` (obstacle detected). The Arduino detects this, rotates the servo motor to 90 degrees to depress a pump, waits 500 ms, returns the servo to 0 degrees, and locks out for 1.5 seconds.

**Why D2 and D9?** Pin D2 reads the touchless hand detection signal. Pin D9 drives the servo motor arm to depress the pump mechanical link.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| IR Sensor | `ir_sensor` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| IR Sensor | VCC | 5V | Power supply (5V) |
| IR Sensor | OUT | D2 | Digital signal connection |
| IR Sensor | GND | GND | Ground reference |
| Servo Motor | VCC | 5V | Power (red wire) |
| Servo Motor | SIG | D9 | Control pin (orange wire) |
| Servo Motor | GND | GND | Ground reference (brown wire) |

## Code
```cpp
#include <Servo.h>

const int IR_PIN    = 2;
const int SERVO_PIN = 9;

Servo dispenserServo;

void setup() {
  pinMode(IR_PIN, INPUT);
  dispenserServo.attach(SERVO_PIN);
  dispenserServo.write(0); // Set to start position
  
  Serial.begin(9600);
  Serial.println("Sanitizer Dispenser Ready");
}

void loop() {
  int handDetected = digitalRead(IR_PIN);
  
  // Active LOW: LOW means hand is in front of the sensor
  if (handDetected == LOW) {
    Serial.println("Hand detected - Dispensing Sanitizer");
    
    // Dispense action: rotate to 90 degrees
    dispenserServo.write(90);
    delay(500); // Wait for liquid to flow
    
    // Retract action: return to 0 degrees
    dispenserServo.write(0);
    
    // Lockout delay: prevent double dispensing
    delay(1500); 
    Serial.println("Ready for next user");
  }
  
  delay(100);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **IR Sensor**, and **Servo Motor** onto the canvas.
2. Connect IR Sensor **VCC** to Arduino **5V**, **OUT** to Arduino **D2**, and **GND** to Arduino **GND**.
3. Connect Servo **VCC** to Arduino **5V**, **SIG** to Arduino **D9**, and **GND** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Click the IR Sensor on the canvas to toggle its "Obstacle" property ON and OFF, and watch the Servo sweep to dispense and return.

## Expected Output

Terminal:
```
Sanitizer Dispenser Ready
Hand detected - Dispensing Sanitizer
Ready for next user
...
```

### Expected Canvas Behavior

| Hand Proximity | Pin D2 State | Servo Pin D9 State | Dispenser Action |
| --- | --- | --- | --- |
| Not present | HIGH (5V) | Parked (0 degrees) | Idle |
| Present (Click) | LOW (0V) | Rotates to 90 -> returns | Dispenses once |

The servo moves through its cycle once per hand detection event.

## Code Walkthrough

| Statement / Block | What It Does |
| --- | --- |
| `dispenserServo.write(90); delay(500);` | Rotates the servo shaft to 90 degrees and waits half a second to simulate pushing a mechanical pump down. |
| `dispenserServo.write(0);` | Moves the shaft back to the home position, allowing the pump to spring back up. |
| `delay(1500)` | Enforces a safety period where the sensor is ignored, preventing excessive dispensing if a user keeps their hand in front of the sensor. |

## Hardware & Safety Concept: Optical Proximity Interference
Active IR sensors rely on detecting reflected infrared light. Shiny surfaces, direct sunlight, or dark materials (which absorb IR light) can cause false triggers or prevent detection. In real-world automatic dispensers, engineers calibrate the sensor's potentiometer to narrow the range to a few centimeters directly under the nozzle.

## Try This! (Challenges)
1. **Double Dose**: Modify the code to make the servo pump twice in rapid succession when a hand is detected.
2. **Speed Pump**: Reduce the pump delay to `300` ms and the return delay to see if the mechanism can operate faster.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo does not move | Signal pin mismatch | Ensure the Servo SIG wire is connected to D9, matching `SERVO_PIN = 9`. |
| Dispenser loops continuously | Hand sensor is dirty or reflection noise | Reposition the sensor or adjust its sensitivity potentiometer on hardware. |

## Mode Notes
These patterns (digital input polling triggering servo write sequences) are supported by MbedO interpreted mode.

## Related Projects
- [22 - Obstacle Print](../beginner/22-obstacle-print.md)
- [36 - Rain Servo Wiper](36-rain-servo-wiper.md)
- [101 - Automatic Gate](../advanced/101-automatic-gate.md)
