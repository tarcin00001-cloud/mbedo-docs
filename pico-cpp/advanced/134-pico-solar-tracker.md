# 134 - Pico Solar Tracker

Build an automated solar tracker that rotates a servo motor to align a solar panel with the brightest light source.

## Goal
Learn how to read two analog light sensors (LDRs) and control a servo motor dynamically to track light gradients (differential light tracking).

## What You Will Build
A single-axis solar tracker model:
- **Left LDR Photoresistor (GP26)**: Measures light on the left side.
- **Right LDR Photoresistor (GP27)**: Measures light on the right side.
- **Servo Motor (GP10)**: Steers the solar panel platform left or right to balance the light levels on both LDR sensors.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LDR Photoresistors | `ldr` | Yes (two LDRs) | Yes |
| Servo Motor | `servo` | Yes | Yes |
| 10k-ohm Resistors | `resistor` | Optional | Yes (voltage dividers) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Left LDR | Signal (Pin 2) | GP26 | Analog input (requires 10k pull-down) |
| Right LDR | Signal (Pin 2) | GP27 | Analog input (requires 10k pull-down) |
| Servo Motor | Signal | GP10 | Rotating platform control |
| Servo Motor | Power / Ground | VBUS (5V) / GND | Servo power rails |
| All Sensors | VCC / GND | 3.3V / GND | Sensor power rails |

## Code
```cpp
#include <Servo.h>

const int LDR_LEFT  = 26;
const int LDR_RIGHT = 27;
const int SERVO_PIN = 10;

Servo trackerServo;

int servoAngle = 90; // Start at center position
const int tolerance = 100; // Analog noise tolerance band

void setup() {
  trackerServo.attach(SERVO_PIN);
  trackerServo.write(servoAngle);
  
  pinMode(LDR_LEFT, INPUT);
  pinMode(LDR_RIGHT, INPUT);
}

void loop() {
  int valLeft  = analogRead(LDR_LEFT);
  int valRight = analogRead(LDR_RIGHT);

  // Calculate the difference between left and right light levels
  int diff = valLeft - valRight;
  int absDiff = diff;
  if (absDiff < 0) { absDiff = -absDiff; }

  // Adjust servo position if difference exceeds tolerance
  if (absDiff > tolerance) {
    if (diff > 0) {
      // More light on left side -> turn servo left
      servoAngle = servoAngle + 3;
      if (servoAngle > 180) { servoAngle = 180; }
    } else {
      // More light on right side -> turn servo right
      servoAngle = servoAngle - 3;
      if (servoAngle < 0) { servoAngle = 0; }
    }
    trackerServo.write(servoAngle);
  }

  delay(50); // Control tracking speed
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two LDRs**, and **Servo Motor** onto the canvas.
2. Connect Left LDR to **GP26**, Right LDR to **GP27**, and Servo to **GP10**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the light level of either LDR up and watch the servo motor rotate to track the light.

## Expected Output

Terminal:
```
Simulation active. Solar light tracking active.
```

## Expected Canvas Behavior
| Left LDR Slider | Right LDR Slider | Servo Motor (GP10) Action |
| --- | --- | --- |
| Equal light | Equal light | Stays at current angle (90°) |
| Bright (Slide Up) | Dark | Rotates left (toward 180°) |
| Dark | Bright (Slide Up) | Rotates right (toward 0°) |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `absDiff > tolerance` | Prevents the servo from oscillating back and forth continuously (jittering) due to minor analog noise. |

## Hardware & Safety Concept: Solar Tracker Efficiency
Solar panels generate maximum electrical power when they are perpendicular to the incoming sun rays. Single-axis solar trackers rotate the panel throughout the day to follow the sun's path from East to West, increasing power generation efficiency by up to 25% compared to fixed-angle panels.

## Try This! (Challenges)
1. **Indicator LEDs**: Add a Red LED on GP15 that turns ON when the platform is rotating, and a Green LED on GP14 when tracking is locked.
2. **Dynamic Step Size**: Modify the code to adjust the servo step speed dynamically based on the size of the light difference (e.g. steering faster if the light difference is large).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo rotates away from the light | Sensor channels swapped | Swap the GP26 and GP27 input pin definitions in your code, or swap the physical LDR positions. |

## Mode Notes
This multi-device analog motor control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [34 - Pico LDR Sensor](../../beginner/34-pico-ldr-sensor.md)
- [46 - Pico Pot Servo](../../beginner/46-pico-pot-servo.md)
- [97 - Pico Joystick Dual Servo](97-pico-joystick-servo.md)
