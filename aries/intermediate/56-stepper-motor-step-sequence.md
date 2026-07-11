# 56 - Stepper Motor Step Sequence (ULN2003)

Drive a 4-phase stepper motor in a continuous rotation sequence using a ULN2003 Darlington array and a loop-free C++ state machine.

## Goal
Learn how stepper motors operate by executing sequential coil activation (wave drive) using digital pins on the VEGA ARIES v3 board without using loops inside C++ code.

## What You Will Build
A stepper motor control system that steps through a 4-step sequence (IN1 to IN4) to spin the motor rotor continuously in one direction at a constant rate.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| ULN2003 Driver Board | `uln2003` | Yes | Yes |
| 28BYJ-48 Stepper Motor | `stepper` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| ULN2003 Driver | IN1 | GPIO 14 | Blue | Phase 1 control line |
| ULN2003 Driver | IN2 | GPIO 15 | Yellow | Phase 2 control line |
| ULN2003 Driver | IN3 | GPIO 13 | Orange | Phase 3 control line |
| ULN2003 Driver | IN4 | GPIO 12 | Green | Phase 4 control line |
| ULN2003 Driver | VCC | 5V | Red | Driver board power |
| ULN2003 Driver | GND | GND | Black | Ground reference |

> **Wiring tip:** The ULN2003 inputs align to ARIES pins GPIO 14, 15, 13, and 12. Ensure these are connected in order to maintain the correct phase stepping sequence.

## Code
```cpp
const int IN1 = 14;
const int IN2 = 15;
const int IN3 = 13;
const int IN4 = 12;
int stepIndex = 0;

void setup() {
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
}

void loop() {
  // Stepper Wave Drive Sequence (Single-Phase Stepping)
  if (stepIndex == 0) {
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, LOW);
  } else if (stepIndex == 1) {
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, HIGH);
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, LOW);
  } else if (stepIndex == 2) {
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, HIGH);
    digitalWrite(IN4, LOW);
  } else if (stepIndex == 3) {
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, HIGH);
  }

  stepIndex = (stepIndex + 1) % 4; // Advance to the next phase in the sequence
  delay(10); // Stepping delay in milliseconds (defines rotation speed)
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **ULN2003 Driver**, and **28BYJ-48 Stepper Motor** components onto the canvas.
2. Wire the ULN2003: **IN1** to **GPIO 14**, **IN2** to **GPIO 15**, **IN3** to **GPIO 13**, **IN4** to **GPIO 12**, **VCC** to **5V**, and **GND** to **GND**.
3. Connect the stepper motor to the ULN2003 output socket.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Stepper Motor Sequence Running.
```

## Expected Canvas Behavior
* The stepper motor rotates smoothly in a clockwise direction.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(IN1, OUTPUT)` | Sets the phase control pin 1 as output. |
| `digitalWrite(IN1, HIGH)` | Activates the first electromagnet coil in the motor stator. |
| `stepIndex = (stepIndex + 1) % 4` | Moves the stepping pointer to the next state, cycling from 0 to 3. |
| `delay(10)` | Limits stepping frequency, preventing the motor from stalling due to inertia. |

## Hardware & Safety Concept
* **Stepper Phase Sequence**: Stepper motors rotate by energizing coils in a specific, overlapping sequence. The wave drive method energizes one coil at a time, pulling the permanent magnet rotor in steps.
* **Inductive Load Protection**: The ULN2003 is a Darlington transistor array with internal clamp diodes. Inductive spikes created when switching stepper coils OFF are safely routed to VCC, protecting the microcontroller.

## Try This! (Challenges)
1. **Direction Swap**: Change the step order to run the motor in reverse (counter-clockwise). (Hint: decrement `stepIndex` or reverse the phase assignments in the sequence).
2. **Full-Step Drive**: Modify the sequence to energize two coils at a time (e.g. IN1+IN2 HIGH, then IN2+IN3 HIGH, etc.) to double the output torque.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor vibrates but does not rotate | Wrong wiring order or too low delay | Verify that pins 14, 15, 13, and 12 are connected in the correct IN1-IN4 order. Also, ensure the delay is not less than 3 ms. |
| ULN2003 chip gets very hot | Current leakage or overload | Ensure VCC is connected to 5V, and the motor is not physically jammed. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [55 - DC Motor Direction Toggle](55-dc-motor-direction-toggle.md)
- [57 - Stepper Motor Speed Regulator (ULN2003)](57-stepper-motor-speed-regulator.md)
