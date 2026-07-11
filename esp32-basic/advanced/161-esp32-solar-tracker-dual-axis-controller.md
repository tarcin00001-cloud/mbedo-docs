# 161 - ESP32 Solar Tracker Dual Axis Controller

Build a dual-axis solar tracker that reads four light-dependent resistors (LDRs) arranged in a grid and adjusts horizontal (Pan) and vertical (Tilt) servos to keep a solar panel facing the brightest light source.

## Goal
Learn how to implement spatial grid averaging to detect directional light vectors, coordinate two servo outputs, and establish mechanical safety limits.

## What You Will Build
Four LDR sensors are arranged in a quadrant grid: Top-Left (GPIO 34), Top-Right (GPIO 35), Bottom-Left (GPIO 32), and Bottom-Right (GPIO 33). Two SG90 servos steer the platform: Horizontal (GPIO 13) and Vertical (GPIO 12). The ESP32 compares light levels and moves the servos to center the platform on the light source.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LDRs (Photoresistors) (4) | `ldr` | Yes | Yes |
| Servos (2) | `servo` | Yes | Yes |
| 10 kΩ Resistors (4) | `resistor` | No | Yes |
| External Power Supply (5V, 2A) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR Top-Left (TL) | Wiper | GPIO34 | Yellow | Quadrant Top-Left |
| LDR Top-Right (TR)| Wiper | GPIO35 | Yellow | Quadrant Top-Right |
| LDR Bottom-Left (BL)| Wiper | GPIO32 | Yellow | Quadrant Bottom-Left |
| LDR Bottom-Right (BR)| Wiper | GPIO33 | Yellow | Quadrant Bottom-Right |
| Servo 1 (Horizontal)| Signal | GPIO13 | Orange | Pan rotation control |
| Servo 2 (Vertical) | Signal | GPIO12 | Blue | Tilt elevation control |
| LDR Dividers (all) | VCC / GND | 3V3 / GND | Red / Black | Shared power rails |
| Servos (all) | VCC / GND | 5V / GND | Red / Black | External power rails |

> **Wiring tip:** The four LDRs must be separated by a vertical cross-divider barrier (baffle) so that light from different angles casts shadows, creating measurable voltage differences between the quadrants.

## Code
```cpp
// Solar Tracker Dual Axis Controller (4x LDR + 2x Servo)
#include <ESP32Servo.h>

const int LDR_TL = 34; // Top-Left LDR
const int LDR_TR = 35; // Top-Right LDR
const int LDR_BL = 32; // Bottom-Left LDR
const int LDR_BR = 33; // Bottom-Right LDR

const int SERVO_H = 13; // Horizontal Servo (Pan)
const int SERVO_V = 12; // Vertical Servo (Tilt)

Servo horizontalServo;
Servo verticalServo;

// Current servo angles
int hAngle = 90;
int vAngle = 90;

// Tracker sensitivity parameters
const int TOLERANCE = 150;      // Minimum LDR difference to trigger movement (raw ADC values)
const int STEP_DELAY_MS = 25;   // Wait time between servo adjustments (controls speed)

// Mechanical safety limits (prevent joint/wire binding)
const int H_LIMIT_MIN = 10;
const int H_LIMIT_MAX = 170;
const int V_LIMIT_MIN = 15;
const int V_LIMIT_MAX = 150;

void setup() {
  Serial.begin(115200);
  
  horizontalServo.attach(SERVO_H);
  verticalServo.attach(SERVO_V);
  
  horizontalServo.write(hAngle);
  verticalServo.write(vAngle);
  
  Serial.println("Dual Axis Solar Tracker online.");
}

void loop() {
  // Read all four quadrants (higher values = brighter light)
  int tl = analogRead(LDR_TL);
  int tr = analogRead(LDR_TR);
  int bl = analogRead(LDR_BL);
  int br = analogRead(LDR_BR);
  
  // Calculate spatial averages
  int avgTop    = (tl + tr) / 2;
  int avgBottom = (bl + br) / 2;
  int avgLeft   = (tl + bl) / 2;
  int avgRight  = (tr + br) / 2;
  
  // Calculate differences
  int diffVert = avgTop - avgBottom;
  int diffHoriz = avgLeft - avgRight;
  
  Serial.print("TL:"); Serial.print(tl);
  Serial.print(" TR:"); Serial.print(tr);
  Serial.print(" BL:"); Serial.print(bl);
  Serial.print(" BR:"); Serial.print(br);
  Serial.print(" | H_Ang:"); Serial.print(hAngle);
  Serial.print(" V_Ang:"); Serial.println(vAngle);
  
  // 1. Adjust Vertical (Tilt) Servo
  if (abs(diffVert) > TOLERANCE) {
    if (avgTop > avgBottom) {
      vAngle--; // Tilt up
    } else {
      vAngle++; // Tilt down
    }
    vAngle = constrain(vAngle, V_LIMIT_MIN, V_LIMIT_MAX);
    verticalServo.write(vAngle);
  }
  
  // 2. Adjust Horizontal (Pan) Servo
  if (abs(diffHoriz) > TOLERANCE) {
    if (avgLeft > avgRight) {
      hAngle++; // Turn left
    } else {
      hAngle--; // Turn right
    }
    hAngle = constrain(hAngle, H_LIMIT_MIN, H_LIMIT_MAX);
    horizontalServo.write(hAngle);
  }
  
  delay(STEP_DELAY_MS); // Pace tracking loops
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, four **LDRs**, and two **Servos** onto the canvas.
2. Wire LDRs to **GPIO34, 35, 32, 33**, Horizontal Servo to **GPIO13**, and Vertical Servo to **GPIO12**.
3. Paste the code and click **Run**.
4. Slide LDR values. Set the Top-Left LDR bright. Watch both servos rotate to steer the panel toward it.

## Expected Output
Serial Monitor:
```
TL:3200 TR:2100 BL:1200 BR:950 | H_Ang:90 V_Ang:90
TL:3200 TR:2100 BL:1200 BR:950 | H_Ang:91 V_Ang:89
```

## Expected Canvas Behavior
* Adjusting the light levels of the four LDR widgets changes the positions of both servo widgets.
* If all LDR widgets are set to the same level (balanced light), the servos stop moving.
* The servos stop automatically when they reach the safety limits (e.g. 10° or 170°).

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `avgTop = (tl + tr) / 2` | Calculates the average light level of the top half of the grid. |
| `abs(diffVert) > TOLERANCE` | Checks if the light difference is large enough to warrant moving the tracker, preventing jitter. |
| `constrain(vAngle, ...)` | Restricts servo travel to prevent mechanical strain or wire binding. |

## Hardware & Safety Concept: Solar Tracker Efficiency
Solar panels generate maximum power when they are perpendicular to the incoming sun rays (angle of incidence = 0°). A static solar panel loses up to 40% of potential energy collection because the sun moves across the sky. Active solar trackers use sensors to keep the panel aligned with the sun throughout the day, increasing energy harvest.

## Try This! (Challenges)
1. **Interactive Status HUD**: Integrate an OLED display (Project 60) showing LDR values and current servo angles.
2. **Night Return Mode**: If all four LDRs read below 400 (indicating night), drive the servos to home positions (90°, 15°) to prepare for the morning sun.
3. **Power Output Logger**: Connect a voltage/current sensor (Project 162) and log energy harvest efficiency metrics.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servos steer away from the light | Servo direction inverted | Swap the increment (`++`) and decrement (`--`) operations in the code |
| Servos oscillate back and forth rapidly | Tolerance setting too low | Increase the `TOLERANCE` variable (e.g. to 200 or 300) |
| Servos move too slowly | Step delay too long | Decrease `STEP_DELAY_MS` to speed up response |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [162 - ESP32 Solar Tracker Efficiency Logger](162-esp32-solar-tracker-efficiency-logger.md) (Next project)
- [133 - ESP32 2-Axis Robotic Arm Joint Controller](133-esp32-2-axis-robotic-arm-joint-controller.md)
- [34 - ESP32 LDR Automatic Dark Detector](../beginner/34-esp32-ldr-automatic-dark-detector.md)
// 161-esp32-solar-tracker-dual-axis-controller.md
