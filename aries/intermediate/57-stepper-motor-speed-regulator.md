# 57 - Stepper Motor Speed Regulator

Regulate the rotational speed of a stepper motor by reading an analog value from a potentiometer and mapping it to the stepping delay in a loop-free C++ state machine.

## Goal
Learn how to read analog inputs using the ADC pins of the VEGA ARIES v3 board and use these readings to dynamically adjust the stepping interval of a stepper motor without using loops inside C++ code.

## What You Will Build
A stepper motor speed controller where rotating the potentiometer changes the delay between step sequences (from 5 ms to 100 ms), dynamically altering the motor's rotational speed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| ULN2003 Driver Board | `uln2003` | Yes | Yes |
| 28BYJ-48 Stepper Motor | `stepper` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| ULN2003 Driver | IN1 | GPIO 14 | Blue | Phase 1 control line |
| ULN2003 Driver | IN2 | GPIO 15 | Yellow | Phase 2 control line |
| ULN2003 Driver | IN3 | GPIO 13 | Orange | Phase 3 control line |
| ULN2003 Driver | IN4 | GPIO 12 | Green | Phase 4 control line |
| ULN2003 Driver | VCC | 5V | Red | Power supply |
| ULN2003 Driver | GND | GND | Black | Ground reference |
| Potentiometer | Pin 1 (VCC) | 3V3 | Red | Analog voltage supply |
| Potentiometer | Pin 2 (Wiper) | ADC0 (GP26) | White | Analog output signal |
| Potentiometer | Pin 3 (GND) | GND | Black | Ground return |

> **Wiring tip:** The potentiometer requires 3.3V power, and its wiper pin connects to ARIES analog input `ADC0` (pin `GP26`). Always use matching grounds.

## Code
```cpp
const int IN1 = 14;
const int IN2 = 15;
const int IN3 = 13;
const int IN4 = 12;
const int POT_PIN = 26; // ADC0 is GP26
int stepIndex = 0;

void setup() {
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
}

void loop() {
  int potValue = analogRead(POT_PIN);
  
  // Map 12-bit potentiometer reading (0 to 4095) to step delay (5 ms to 100 ms)
  int stepDelay = 5 + (potValue * 95 / 4095);

  // Stepper Wave Drive Sequence
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

  stepIndex = (stepIndex + 1) % 4; // Cycle sequence
  delay(stepDelay); // Dynamic pause based on potentiometer position
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **ULN2003 Driver**, **28BYJ-48 Stepper Motor**, and **Potentiometer** components onto the canvas.
2. Wire the potentiometer: **Pin 1** to **3V3**, **Wiper** to **ADC0 (GP26)**, and **Pin 3** to **GND**.
3. Wire the ULN2003: **IN1-IN4** to **GPIO 14, 15, 13, 12**, **VCC** to **5V**, and **GND** to **GND**. Connect the motor to the driver.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Potentiometer Reading: 2048, Step Delay: 52 ms
```

## Expected Canvas Behavior
* Adjusting the potentiometer dial in the simulator changes the speed of the stepper motor's visual rotation. Turn it clockwise to slow down (increases delay), counter-clockwise to speed up (decreases delay).

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `analogRead(POT_PIN)` | Reads the analog voltage on GP26 (returns 0 to 4095). |
| `5 + (potValue * 95 / 4095)` | Maps the 12-bit ADC value to a safe step-delay range (5 to 100 ms). |
| `digitalWrite(IN1, HIGH)` | Drives the active stepper phase line HIGH. |
| `delay(stepDelay)` | Pauses the execution loop for the mapped delay value, regulating the step rate. |

## Hardware & Safety Concept
* **Stepper Resonance and Stall**: Stepper motors have a maximum start-stop frequency. If you step them too quickly (e.g. delay < 2 ms), the magnetic fields switch faster than the rotor can physically move, causing the motor to stall, vibrate, and hum without spinning.
* **ADC Voltage Limits**: The VEGA ARIES v3 microcontroller's ADC pin logic level is 3.3V. Applying a voltage higher than 3.3V to the ADC pins can destroy the ADC multiplexer inside the SoC.

## Try This! (Challenges)
1. **Change Step Angle**: Change the mapped delay range to `10 ms` to `150 ms` to see how the motor responds to extremely slow stepping rates.
2. **Reverse Switch**: Add a switch or wire connection to check a GPIO input; reverse the rotation direction if the input state changes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Potentiometer has no effect on speed | Wired to wrong analog pin | Verify that the wiper is connected specifically to ADC0 (GP26), not ADC1 or ADC2. |
| Motor stalls at lowest delay | Delay step value is too short | Increase the offset in the delay mapping formula (e.g. change 5 to 10). |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [56 - Stepper Motor Step Sequence (ULN2003)](56-stepper-motor-step-sequence.md)
- [68 - Stepper Motor Position Log (LCD display)](68-stepper-motor-position-log.md)
