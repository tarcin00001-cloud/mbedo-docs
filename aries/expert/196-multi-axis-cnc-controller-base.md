# 196 - Multi-Axis CNC Controller Base

Develop a multi-axis CNC controller coordinate base that drives three independent stepper motors (representing X, Y, and Z cartesian coordinate axes) using ULN2003 driver chips, executing coordinated linear movements through a loop-free C++ state machine.

## Goal
Learn how to manage multiple stepper motors simultaneously, write coordinate tracking states using scalar variables rather than arrays, control unipolar stepping sequences, and coordinate multi-axis motion curves without blocking execution.

## What You Will Build
A cartesian plotter driver. The ARIES v3 board controls three stepper motors (X-axis on GP2-GP5, Y-axis on GP6-GP9, and Z-axis on GP10-GP13). The controller executes a coordinated motion path by reading target coordinates from a switch-case state machine. The board calculates step directions and cycles each stepper motor to its target coordinates, printing coordinates to the Serial Monitor at each step.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| 3x 28BYJ-48 Stepper Motors | `stepper` | Yes | Yes |
| 3x ULN2003 Stepper Drivers | `stepper_driver` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| ULN2003 X (Driver 1) | IN1 | GPIO 2 | Blue | X-axis coil A |
| ULN2003 X (Driver 1) | IN2 | GPIO 3 | Yellow | X-axis coil B |
| ULN2003 X (Driver 1) | IN3 | GPIO 4 | Green | X-axis coil C |
| ULN2003 X (Driver 1) | IN4 | GPIO 5 | White | X-axis coil D |
| ULN2003 Y (Driver 2) | IN1 | GPIO 6 | Blue | Y-axis coil A |
| ULN2003 Y (Driver 2) | IN2 | GPIO 7 | Yellow | Y-axis coil B |
| ULN2003 Y (Driver 2) | IN3 | GPIO 8 | Green | Y-axis coil C |
| ULN2003 Y (Driver 2) | IN4 | GPIO 9 | White | Y-axis coil D |
| ULN2003 Z (Driver 3) | IN1 | GPIO 10 | Blue | Z-axis coil A |
| ULN2003 Z (Driver 3) | IN2 | GPIO 11 | Yellow | Z-axis coil B |
| ULN2003 Z (Driver 3) | IN3 | GPIO 12 | Green | Z-axis coil C |
| ULN2003 Z (Driver 3) | IN4 | GPIO 13 | White | Z-axis coil D |
| All Drivers | VCC | 5V | Red | Motor power supply |
| All Drivers | GND | GND | Black | Shared ground reference |

> **Wiring tip:** Stepper motors draw substantial inductive spikes during coil switching. Never power three stepper motors directly from the ARIES 5V power supply on hardware, as this will reset the microcontroller. Use an external 5V–12V DC power supply for the ULN2003 drivers while connecting all grounds together.

## Code
```cpp
// 196 - Multi-Axis CNC Controller Base
#include <Arduino.h>

// X-Axis Pins
const int X_IN1 = 2;
const int X_IN2 = 3;
const int X_IN3 = 4;
const int X_IN4 = 5;

// Y-Axis Pins
const int Y_IN1 = 6;
const int Y_IN2 = 7;
const int Y_IN3 = 8;
const int Y_IN4 = 9;

// Z-Axis Pins
const int Z_IN1 = 10;
const int Z_IN2 = 11;
const int Z_IN3 = 12;
const int Z_IN4 = 13;

// Coordinate State variables (scalar variables - NO arrays allowed)
int posX = 0;
int posY = 0;
int posZ = 0;

int targetX = 80;   // Target coordinate 1
int targetY = 120;
int targetZ = 40;

int stepX = 0;      // Current sequence state index (0 to 3)
int stepY = 0;
int stepZ = 0;

int pathStage = 0;  // 0 = Move to Coord 1, 1 = Move to Coord 2, 2 = Move to Coord 3, 3 = Home
int pauseTicks = 0;
int logTimer = 0;

void setup() {
  Serial.begin(115200);

  // Configure output pins
  pinMode(X_IN1, OUTPUT); pinMode(X_IN2, OUTPUT); pinMode(X_IN3, OUTPUT); pinMode(X_IN4, OUTPUT);
  pinMode(Y_IN1, OUTPUT); pinMode(Y_IN2, OUTPUT); pinMode(Y_IN3, OUTPUT); pinMode(Y_IN4, OUTPUT);
  pinMode(Z_IN1, OUTPUT); pinMode(Z_IN2, OUTPUT); pinMode(Z_IN3, OUTPUT); pinMode(Z_IN4, OUTPUT);

  // Set all coils LOW initially
  digitalWrite(X_IN1, LOW); digitalWrite(X_IN2, LOW); digitalWrite(X_IN3, LOW); digitalWrite(X_IN4, LOW);
  digitalWrite(Y_IN1, LOW); digitalWrite(Y_IN2, LOW); digitalWrite(Y_IN3, LOW); digitalWrite(Y_IN4, LOW);
  digitalWrite(Z_IN1, LOW); digitalWrite(Z_IN2, LOW); digitalWrite(Z_IN3, LOW); digitalWrite(Z_IN4, LOW);

  Serial.println("CNC Multi-Axis Controller Online.");
  Serial.print("Target 1: (");
  Serial.print(targetX); Serial.print(", ");
  Serial.print(targetY); Serial.print(", ");
  Serial.print(targetZ); Serial.println(")");
}

void loop() {
  // If we are pausing between coordinates
  if (pauseTicks > 0) {
    pauseTicks--;
    delay(10);
    return;
  }

  // Flag to track if any axis moved during this loop cycle
  int axisActive = 0;

  // --- X-AXIS CONTROLLER STATE ---
  if (posX != targetX) {
    axisActive = 1;
    int dirX = (targetX > posX) ? 1 : -1;
    posX += dirX;
    
    // Increment stepper state index (0-3)
    stepX = (stepX + dirX + 4) % 4;
    
    // Apply 4-step sequence
    if (stepX == 0) {
      digitalWrite(X_IN1, HIGH); digitalWrite(X_IN2, LOW);  digitalWrite(X_IN3, LOW);  digitalWrite(X_IN4, LOW);
    } else if (stepX == 1) {
      digitalWrite(X_IN1, LOW);  digitalWrite(X_IN2, HIGH); digitalWrite(X_IN3, LOW);  digitalWrite(X_IN4, LOW);
    } else if (stepX == 2) {
      digitalWrite(X_IN1, LOW);  digitalWrite(X_IN2, LOW);  digitalWrite(X_IN3, HIGH); digitalWrite(X_IN4, LOW);
    } else {
      digitalWrite(X_IN1, LOW);  digitalWrite(X_IN2, LOW);  digitalWrite(X_IN3, LOW);  digitalWrite(X_IN4, HIGH);
    }
  }

  // --- Y-AXIS CONTROLLER STATE ---
  if (posY != targetY) {
    axisActive = 1;
    int dirY = (targetY > posY) ? 1 : -1;
    posY += dirY;
    
    stepY = (stepY + dirY + 4) % 4;
    
    if (stepY == 0) {
      digitalWrite(Y_IN1, HIGH); digitalWrite(Y_IN2, LOW);  digitalWrite(Y_IN3, LOW);  digitalWrite(Y_IN4, LOW);
    } else if (stepY == 1) {
      digitalWrite(Y_IN1, LOW);  digitalWrite(Y_IN2, HIGH); digitalWrite(Y_IN3, LOW);  digitalWrite(Y_IN4, LOW);
    } else if (stepY == 2) {
      digitalWrite(Y_IN1, LOW);  digitalWrite(Y_IN2, LOW);  digitalWrite(Y_IN3, HIGH); digitalWrite(Y_IN4, LOW);
    } else {
      digitalWrite(Y_IN1, LOW);  digitalWrite(Y_IN2, LOW);  digitalWrite(Y_IN3, LOW);  digitalWrite(Y_IN4, HIGH);
    }
  }

  // --- Z-AXIS CONTROLLER STATE ---
  if (posZ != targetZ) {
    axisActive = 1;
    int dirZ = (targetZ > posZ) ? 1 : -1;
    posZ += dirZ;
    
    stepZ = (stepZ + dirZ + 4) % 4;
    
    if (stepZ == 0) {
      digitalWrite(Z_IN1, HIGH); digitalWrite(Z_IN2, LOW);  digitalWrite(Z_IN3, LOW);  digitalWrite(Z_IN4, LOW);
    } else if (stepZ == 1) {
      digitalWrite(Z_IN1, LOW);  digitalWrite(Z_IN2, HIGH); digitalWrite(Z_IN3, LOW);  digitalWrite(Z_IN4, LOW);
    } else if (stepZ == 2) {
      digitalWrite(Z_IN1, LOW);  digitalWrite(Z_IN2, LOW);  digitalWrite(Z_IN3, HIGH); digitalWrite(Z_IN4, LOW);
    } else {
      digitalWrite(Z_IN1, LOW);  digitalWrite(Z_IN2, LOW);  digitalWrite(Z_IN3, LOW);  digitalWrite(Z_IN4, HIGH);
    }
  }

  // --- PATH STATE MACHINE ---
  // If all axes have reached their target
  if (axisActive == 0) {
    Serial.print("Target Reached: (");
    Serial.print(posX); Serial.print(", ");
    Serial.print(posY); Serial.print(", ");
    Serial.print(posZ); Serial.println(")");
    
    pauseTicks = 100; // Pause for 1 second (100 * 10 ms delay)
    pathStage = (pathStage + 1) % 4;

    // Load next target coordinate using scalar switch logic (NO arrays)
    if (pathStage == 0) {
      targetX = 80;  targetY = 120; targetZ = 40;
    } else if (pathStage == 1) {
      targetX = 0;   targetY = 0;   targetZ = 0;
    } else if (pathStage == 2) {
      targetX = -100; targetY = 80;  targetZ = -20;
    } else if (pathStage == 3) {
      targetX = 0;   targetY = 0;   targetZ = 0;
    }
    
    Serial.print("Next Target: (");
    Serial.print(targetX); Serial.print(", ");
    Serial.print(targetY); Serial.print(", ");
    Serial.print(targetZ); Serial.println(")");
  }

  // Serial status log output every ~500 ms (50 * 10 ms delay)
  logTimer++;
  if (logTimer >= 50) {
    logTimer = 0;
    Serial.print("Position: X=");
    Serial.print(posX);
    Serial.print(" Y=");
    Serial.print(posY);
    Serial.print(" Z=");
    Serial.println(posZ);
  }

  delay(10); // Dictates stepper motor speed limit
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board and **3x Stepper Motor** components onto the canvas.
2. Wire X-Axis pins: IN1-IN4 to **GPIO 2, 3, 4, 5**; Y-Axis pins: IN1-IN4 to **GPIO 6, 7, 8, 9**; Z-Axis pins: IN1-IN4 to **GPIO 10, 11, 12, 13**.
3. Wire all motor power lines to **5V** and grounds to **GND**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute.
7. Observe the coordinates changing in the Serial Monitor and the stepper widget graphics rotating in synchronization.

## Expected Output
Serial Monitor:
```
CNC Multi-Axis Controller Online.
Target 1: (80, 120, 40)
Position: X=42 Y=42 Z=40
Position: X=50 Y=92 Z=40
Position: X=80 Y=120 Z=40
Target Reached: (80, 120, 40)
Next Target: (0, 0, 0)
Position: X=50 Y=50 Z=20
```

## Expected Canvas Behavior
* On startup, X, Y, and Z stepper motors start turning.
* The motors rotate at slightly different speeds depending on the distance they must travel (X travels 80 steps, Y travels 120, Z travels 40).
* Once all three motors stop moving, the Serial Monitor prints `Target Reached` and loads the next set of coordinates, causing the motors to reverse direction.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(X_IN1, OUTPUT)` | Defines digital pins for Stepper X driving coils. |
| `posX != targetX` | Checks if the current cartesian coordinate matches target. |
| `dirX = targetX > posX ? 1 : -1` | Calculates direction multiplier (clockwise or counter-clockwise). |
| `stepX = (stepX + dirX + 4) % 4` | Cycles the step index through unipolar sequence phases (0 to 3). |
| `digitalWrite(X_IN1, HIGH)` | Activates specific electromagnetic coil to pull rotor to position. |
| `axisActive = 0` | Resets active movement flag to check coordinates completion. |

## Hardware & Safety Concept
* **Unipolar Stepper Motors:** The 28BYJ-48 is a unipolar stepper motor containing 4 coils. Activating coils in sequence (e.g. A, B, C, D) pulls the rotor in 5.625° increments. The internal gear reduction ratio of 64:1 results in 2048 steps per full output shaft revolution.
* **Coil De-energizing Safety:** If the controller stops stepper movements but leaves coil pins HIGH, current will continuously flow through the stator coil. This causes resistive heating, causing the stepper driver and motor to become extremely hot. Production code pulls all pins LOW when idle to prevent thermal issues.

## Try This! (Challenges)
1. **Half-Step Execution:** Modify the coil driver sequence from 4-step full stepping to 8-step half-stepping (activating two coils concurrently, e.g., AB, B, BC, C) to double step resolution.
2. **Axis Limit Switches:** Connect three push buttons representing X, Y, and Z mechanical limit switches on GPIO 14, 15, and 16. If any button is pressed, halt the corresponding motor immediately.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Stepper motor vibrates but does not rotate | Coil wiring sequences out of order | Verify IN1–IN4 are wired in exact order to GP2-GP5 (X), GP6-GP9 (Y), GP10-GP13 (Z). |
| Board resets when motor turns | Excess current draw | Power the stepper drivers using an external 5V/12V power supply. Shared GND is mandatory. |
| Speed is extremely slow | Delay timing too high | Decrease the loop delay from 10 ms to 4 ms (do not go below 2 ms as stepper rotors cannot spin that fast). |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [56 - Stepper Motor Step Sequence](../intermediate/56-stepper-motor-step-sequence.md)
- [68 - Stepper Motor Position Log](../intermediate/68-stepper-motor-position-log.md)
