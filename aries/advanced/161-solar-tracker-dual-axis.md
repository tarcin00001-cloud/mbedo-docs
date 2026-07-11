# 161 - Solar Tracker Dual Axis Controller

Automatically orient a solar panel toward the brightest light source using four Light-Dependent Resistors (LDRs) at the corners of the panel and two servo motors controlling elevation and azimuth axes.

## Goal
Learn how to read four analog LDR sensors on ADC0–ADC3, compare quadrant brightness values, and drive two servo motors to eliminate the light-level differential — implementing a closed-loop dual-axis solar tracking system.

## What You Will Build
Four LDRs are mounted at the top-left, top-right, bottom-left, and bottom-right corners of a cross-shaped divider on the panel frame. Their analog voltages are read on ADC0 (GP26), ADC1 (GP27), ADC2 (GP28), and ADC3 (GP29). A horizontal servo on GPIO 13 controls azimuth (left/right) and a vertical servo on GPIO 12 controls elevation (up/down). The board computes average brightness differences between left/right and top/bottom pairs and nudges each servo toward the brighter side until the panel is balanced.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| LDR / Photoresistor × 4 | `ldr` | Yes | Yes |
| 10 kΩ Resistor × 4 (pull-down) | `resistor` | No | Yes |
| Servo Motor (Azimuth) | `servo` | Yes | Yes |
| Servo Motor (Elevation) | `servo` | Yes | Yes |
| Solar Panel Frame (DIY) | — | No | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR Top-Left (TL) | Wiper | ADC0 / GP26 | Yellow | Voltage divider with 10 kΩ pull-down |
| LDR Top-Right (TR) | Wiper | ADC1 / GP27 | Orange | Voltage divider with 10 kΩ pull-down |
| LDR Bottom-Left (BL) | Wiper | ADC2 / GP28 | Green | Voltage divider with 10 kΩ pull-down |
| LDR Bottom-Right (BR) | Wiper | ADC3 / GP29 | Blue | Voltage divider with 10 kΩ pull-down |
| All LDRs | One leg | 3V3 | Red | LDR to supply |
| All Pull-down Resistors | Other leg | GND | Black | 10 kΩ resistor from wiper to GND |
| Azimuth Servo | Signal | GPIO 13 | White | PWM control signal |
| Azimuth Servo | VCC | 5V | Red | Servo power |
| Azimuth Servo | GND | GND | Black | Servo ground |
| Elevation Servo | Signal | GPIO 12 | White | PWM control signal |
| Elevation Servo | VCC | 5V | Red | Servo power |
| Elevation Servo | GND | GND | Black | Servo ground |

> **Wiring tip:** Mount the four LDRs symmetrically at the corners of a cross-shaped divider (a + shaped card) attached to the front of the panel. The divider casts a shadow on one or more LDRs whenever the panel is off-angle; this creates a brightness imbalance that the servo logic corrects. Use a separate 5 V supply for both servo motors to avoid voltage droop on the ARIES 5 V rail.

## Code
```cpp
// Solar Tracker Dual Axis Controller
// LDRs: ADC0(TL)=GP26, ADC1(TR)=GP27, ADC2(BL)=GP28, ADC3(BR)=GP29
// Servos: Azimuth=GPIO13, Elevation=GPIO12

#include <Servo.h>

#define LDR_TL  26   // ADC0 Top-Left
#define LDR_TR  27   // ADC1 Top-Right
#define LDR_BL  28   // ADC2 Bottom-Left
#define LDR_BR  29   // ADC3 Bottom-Right

#define SERVO_AZ  13   // Azimuth (left/right)
#define SERVO_EL  12   // Elevation (up/down)

// Tolerance band: difference below this value triggers no movement
const int TOLERANCE = 50;

// Step size in degrees per loop cycle
const int STEP_DEG = 1;

// Servo angle limits
const int AZ_MIN = 0;
const int AZ_MAX = 180;
const int EL_MIN = 10;
const int EL_MAX = 170;

Servo servoAz;
Servo servoEl;

int angleAz = 90;
int angleEl = 90;
int tl = 0;
int tr = 0;
int bl = 0;
int br = 0;
int avgLeft   = 0;
int avgRight  = 0;
int avgTop    = 0;
int avgBottom = 0;
int diffH = 0;
int diffV = 0;

void setup() {
  Serial.begin(115200);
  servoAz.attach(SERVO_AZ);
  servoEl.attach(SERVO_EL);

  servoAz.write(angleAz);
  servoEl.write(angleEl);

  Serial.println("=== Dual-Axis Solar Tracker ===");
  Serial.println("Servos centred at 90 degrees. Tracking begins...");
  delay(1000);
}

void loop() {
  tl = analogRead(LDR_TL);
  tr = analogRead(LDR_TR);
  bl = analogRead(LDR_BL);
  br = analogRead(LDR_BR);

  avgLeft   = (tl + bl) / 2;
  avgRight  = (tr + br) / 2;
  avgTop    = (tl + tr) / 2;
  avgBottom = (bl + br) / 2;

  diffH = avgLeft - avgRight;
  diffV = avgTop  - avgBottom;

  // Azimuth: move toward brighter side (left/right)
  if (diffH > TOLERANCE) {
    angleAz = angleAz - STEP_DEG;  // Rotate left (toward left LDRs)
  } else if (diffH < -TOLERANCE) {
    angleAz = angleAz + STEP_DEG;  // Rotate right
  }

  // Elevation: move toward brighter side (up/down)
  if (diffV > TOLERANCE) {
    angleEl = angleEl + STEP_DEG;  // Tilt up
  } else if (diffV < -TOLERANCE) {
    angleEl = angleEl - STEP_DEG;  // Tilt down
  }

  // Clamp angles to safe limits
  if (angleAz < AZ_MIN) angleAz = AZ_MIN;
  if (angleAz > AZ_MAX) angleAz = AZ_MAX;
  if (angleEl < EL_MIN) angleEl = EL_MIN;
  if (angleEl > EL_MAX) angleEl = EL_MAX;

  servoAz.write(angleAz);
  servoEl.write(angleEl);

  Serial.print("TL="); Serial.print(tl);
  Serial.print(" TR="); Serial.print(tr);
  Serial.print(" BL="); Serial.print(bl);
  Serial.print(" BR="); Serial.print(br);
  Serial.print(" | Az="); Serial.print(angleAz);
  Serial.print("° El="); Serial.print(angleEl);
  Serial.println("°");

  delay(100);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board onto the canvas.
2. Drag four **LDR** components and connect their wipers to **ADC0/GP26**, **ADC1/GP27**, **ADC2/GP28**, and **ADC3/GP29** respectively.
3. Drag two **Servo** components; connect Signal pins to **GPIO 13** (azimuth) and **GPIO 12** (elevation).
4. Paste the code into the editor.
5. Select **Interpreted Mode** from the simulation dropdown.
6. Click **Run**.
7. Adjust the four LDR light-level sliders independently to simulate off-angle sunlight; the servo position indicators should move toward the brighter quadrant.
8. When all four LDR sliders are equal, both servos should hold their current position.

## Expected Output
Serial Monitor:
```
=== Dual-Axis Solar Tracker ===
Servos centred at 90 degrees. Tracking begins...
TL=2100 TR=900 BL=2050 BR=950 | Az=89° El=90°
TL=2100 TR=900 BL=2050 BR=950 | Az=88° El=90°
TL=2050 TR=950 BL=2030 BR=980 | Az=87° El=90°
TL=1500 TR=1500 BL=1500 BR=1500 | Az=87° El=90°
```

## Expected Canvas Behavior
* Both servo widgets update their angle indicator every 100 ms.
* When the left-side LDRs report more light than the right-side LDRs, the azimuth servo moves left by 1° per cycle.
* When all four LDRs are balanced within the tolerance band, both servos hold position.
* The Serial Monitor prints all four raw LDR values and both servo angles continuously.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `#include <Servo.h>` | Imports the Servo library to generate PWM signals for the servo motors. |
| `TOLERANCE = 50` | Dead-band threshold; prevents jitter when the panel is nearly centred. |
| `STEP_DEG = 1` | Move each servo by 1° per loop cycle for smooth, slow tracking. |
| `analogRead(LDR_TL)` | Reads the 12-bit ADC value from the top-left LDR voltage divider. |
| `avgLeft = (tl + bl) / 2` | Averages the two left LDRs to get overall left-side brightness. |
| `diffH = avgLeft - avgRight` | Horizontal brightness differential drives azimuth servo direction. |
| `diffV = avgTop - avgBottom` | Vertical brightness differential drives elevation servo direction. |
| `if (angleAz < AZ_MIN) angleAz = AZ_MIN` | Clamps servo angle to prevent mechanical over-travel damage. |
| `servoAz.write(angleAz)` | Sends the updated PWM position command to the azimuth servo. |

## Hardware & Safety Concept
* **Closed-Loop Position Control**: This is a proportional bang-bang controller with a dead-band. The system continuously senses light imbalance and drives the output (servo angle) in the correcting direction until the error falls within the tolerance zone. Adding a proportional gain (multiplying diffH by a factor before stepping) would create a true P-controller with faster convergence.
* **Cross-Shadow Method**: The + shaped divider between the four LDRs is critical. Without it, diffuse sky illumination would prevent any meaningful differential even when the panel is far off-angle. The divider ensures that only direct-beam light hits each sensor quadrant, providing a strong, directional signal.
* **Servo Power Isolation**: Two servo motors under load can each draw 500 mA or more. Always power servos from a dedicated 5 V supply rather than the ARIES board's onboard regulator, which is typically limited to 300–500 mA total.

## Try This! (Challenges)
1. **Sunrise/Sunset Reset**: Use the average of all four LDR readings as a light level indicator. When the average drops below a threshold (simulating night), move both servos to a park position (east-facing, low elevation) ready for the next morning.
2. **Speed-Proportional Tracking**: Instead of a fixed `STEP_DEG = 1`, make the step size proportional to the magnitude of `diffH` and `diffV`, so the panel slews quickly when far off-angle and slows down near the target, reducing overshoot.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servos oscillate continuously | Tolerance too small or LDR noise | Increase `TOLERANCE` from 50 to 80–150 to widen the dead-band. |
| Only one servo moves | ADC channels swapped or one LDR open-circuit | Verify all four LDR wipers are on their correct ADC pins; check pull-down resistors. |
| Servo slams to limit immediately | Diffuse light without a shadow divider, causing all LDRs to read near-equal values | Check `avgLeft` vs `avgRight` in Serial Monitor; if they are equal the servo should not move — verify the tolerance condition. |
| Servo angles stay at 90° always | Servo library not attached or wrong pin | Confirm `servoAz.attach(SERVO_AZ)` and `servoEl.attach(SERVO_EL)` are in `setup()`. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [162 - Solar Tracker Efficiency Logger](162-solar-tracker-efficiency-logger.md)
- [160 - Ultrasonic Wind Anemometer](160-ultrasonic-wind-anemometer.md)
- [163 - Water Quality Station](163-water-quality-station.md)
