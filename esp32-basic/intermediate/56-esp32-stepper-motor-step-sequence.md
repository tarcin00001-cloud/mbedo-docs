# 56 - ESP32 Stepper Motor Step Sequence

Drive a 28BYJ-48 unipolar stepper motor through its full 4-step half-wave sequence using a ULN2003 driver board, advancing the shaft one step at a time.

## Goal
Learn how a unipolar stepper motor works, how the ULN2003 driver board interfaces its high-current coils to ESP32 GPIOs, and how to implement the 4-step energisation sequence in code.

## What You Will Build
A ULN2003 stepper driver with a 28BYJ-48 motor. Four ESP32 GPIOs (18, 19, 21, 22) energise the four coil phases in sequence. The motor rotates 512 full step sequences (one full revolution) forward, then reverses.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| ULN2003 Stepper Driver Board | `stepper_driver` | Yes | Yes |
| 28BYJ-48 Stepper Motor | `stepper_motor` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| ULN2003 Board | IN1 | GPIO18 | Yellow | Coil phase 1 |
| ULN2003 Board | IN2 | GPIO19 | Green | Coil phase 2 |
| ULN2003 Board | IN3 | GPIO21 | Blue | Coil phase 3 |
| ULN2003 Board | IN4 | GPIO22 | Purple | Coil phase 4 |
| ULN2003 Board | VCC (+) | 5V (Vin) | Red | Motor coil supply — must be 5 V |
| ULN2003 Board | GND (−) | GND | Black | Shared ground with ESP32 |

> **Wiring tip:** The 28BYJ-48 connects to the ULN2003 board via its 5-pin JST connector — plug it in directly. The ULN2003 board requires **5 V** on VCC for proper coil drive current (~200 mA). Use the ESP32's Vin pin. Always de-energise the coils (all pins LOW) when the motor is at rest to prevent overheating.

## Code
```cpp
// Stepper Motor 4-Step Sequence (half-wave drive)
const int IN1 = 18;
const int IN2 = 19;
const int IN3 = 21;
const int IN4 = 22;

// Full 4-step wave drive sequence
const int STEPS[4][4] = {
  {1, 0, 0, 0},   // Step 1: Coil A
  {0, 1, 0, 0},   // Step 2: Coil B
  {0, 0, 1, 0},   // Step 3: Coil C
  {0, 0, 0, 1}    // Step 4: Coil D
};

const int STEP_DELAY = 2;           // ms per step
const int STEPS_PER_REV = 2048;     // 28BYJ-48 gear ratio produces 2048 steps/rev

void stepMotor(int step) {
  digitalWrite(IN1, STEPS[step][0]);
  digitalWrite(IN2, STEPS[step][1]);
  digitalWrite(IN3, STEPS[step][2]);
  digitalWrite(IN4, STEPS[step][3]);
}

void allOff() {
  digitalWrite(IN1, LOW); digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW); digitalWrite(IN4, LOW);
}

void rotate(int steps, bool forward) {
  for (int i = 0; i < steps; i++) {
    int seq = forward ? (i % 4) : (3 - (i % 4));
    stepMotor(seq);
    delay(STEP_DELAY);
  }
  allOff();   // De-energise coils after rotation
}

void setup() {
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT);
  allOff();

  Serial.begin(115200);
  Serial.println("Stepper Motor Sequence ready.");
}

void loop() {
  Serial.println("Rotating FORWARD 1 rev...");
  rotate(STEPS_PER_REV, true);
  delay(1000);

  Serial.println("Rotating REVERSE 1 rev...");
  rotate(STEPS_PER_REV, false);
  delay(1000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **ULN2003 Driver**, and **Stepper Motor** onto the canvas.
2. Connect IN1–IN4 to **GPIO18, 19, 21, 22** respectively.
3. Paste the code and click **Run**.
4. Observe the stepper widget rotating one full revolution forward then reverse.

## Expected Output
Serial Monitor:
```
Stepper Motor Sequence ready.
Rotating FORWARD 1 rev...
Rotating REVERSE 1 rev...
Rotating FORWARD 1 rev...
```

## Expected Canvas Behavior
* Stepper widget rotates smoothly one full revolution clockwise in ~4 seconds.
* Pauses 1 second.
* Rotates one full revolution counter-clockwise.
* Cycle repeats.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `STEPS[4][4]` | A 4×4 truth table: each row is one step, each column is one coil pin state. |
| `stepMotor(step)` | Writes one row of the truth table to the four output pins. |
| `i % 4` | Cycles the step index 0→1→2→3→0 as the loop counter increments. |
| `3 - (i % 4)` | Reverses the sequence (3→2→1→0) for reverse rotation. |
| `allOff()` | Sets all four coil pins LOW after rotation — prevents heat buildup in the coils. |

## Hardware & Safety Concept: Unipolar Stepper Motor Operation
A unipolar stepper motor has four coil windings (or two bifilar windings). Energising one coil creates a magnetic field that attracts the rotor's permanent magnet to align with it. Sequentially energising adjacent coils causes the rotor to step in discrete angular increments. The 28BYJ-48 has a 1/64 gear reduction, giving an effective step angle of 360° ÷ 2048 steps ≈ **0.176° per step** — excellent resolution for slow, precise positioning. The ULN2003 contains Darlington transistor pairs that amplify the ESP32's 3.3 V, 20 mA logic signals into 5 V, 200 mA coil drive currents. Always de-energise the coils at rest — continuous coil energisation at standstill causes resistive heating without useful work.

## Try This! (Challenges)
1. **Half-step mode**: Use an 8-step sequence (alternating single and dual coil energisation) for twice the resolution and smoother motion.
2. **Degrees function**: Write a `rotateDeg(float degrees)` function that converts degrees to steps automatically.
3. **Potentiometer position**: Read GPIO 34 and map it to a target step position — move the motor to that position and hold.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor vibrates but does not rotate | Step sequence in wrong order | Verify the STEPS array: each row should have exactly one HIGH |
| Motor runs hot when idle | Coils not de-energised | Confirm `allOff()` is called after each `rotate()` |
| Motor skips steps | STEP_DELAY too short | Increase to 3–5 ms per step |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [57 - ESP32 Stepper Motor Speed Controller](57-esp32-stepper-motor-speed-controller.md)
- [53 - ESP32 DC Motor Start/Stop](53-esp32-dc-motor-start-stop.md)
- [68 - ESP32 Stepper Motor Position Log](68-esp32-stepper-motor-position-log.md)
