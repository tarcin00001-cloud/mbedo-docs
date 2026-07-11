# 45 - Potentiometer Servo Sweep (GPIO PWM)

Control the angle of a hobby servo motor using a potentiometer connected to the VEGA ARIES v3 board.

## Goal
Understand how hobby servo motors are controlled using Pulse Width Modulation (PWM), write code to generate manual timing pulses at 50 Hz without external libraries, and map analog inputs to precise pulse durations.

## What You Will Build
A potentiometer is connected to analog input pin `ADC0` (`GP26`), and a hobby servo motor (e.g., SG90) is connected to digital pin `GPIO 13`. As the potentiometer is rotated, the board reads the value and outputs a corresponding control pulse (between 1000 and 2000 microseconds) every 20 milliseconds to rotate the servo arm between 0° and 180°.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| Hobby Servo Motor | `servo` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | VCC | 3.3V | Red | Power connection |
| Potentiometer | Output / Signal | ADC0 (GP26) | Yellow | Analog input signal |
| Potentiometer | GND | GND | Black | Ground connection |
| Hobby Servo | VCC (Red/Orange) | 5V | Red | Power connection (servos require 5V) |
| Hobby Servo | Signal (Yellow/Orange) | GPIO 13 | Yellow | Servo control pulse signal |
| Hobby Servo | GND (Brown/Black) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Servos draw significant current while rotating. Always power the servo from the ARIES board's 5V rail (or an external 5V supply) and ensure a common ground is shared with the board. Never power it from the 3.3V pin.

## Code
```cpp
// Potentiometer Servo Control - VEGA ARIES v3
const int POT_PIN = GP26;   // ADC0
const int SERVO_PIN = 13;   // GPIO 13

void setup() {
  // Configure the servo control pin as an output
  pinMode(SERVO_PIN, OUTPUT);
}

void loop() {
  // Read raw potentiometer analog value
  int potVal = analogRead(POT_PIN);
  
  // Map 12-bit ADC value (0-4095) to servo pulse width (1000 - 2000 microseconds)
  // 1000 us = 0 degrees, 1500 us = 90 degrees, 2000 us = 180 degrees
  int pulseWidth = map(potVal, 0, 4095, 1000, 2000);
  
  // Start the 20ms duty cycle frame: Drive signal pin HIGH
  digitalWrite(SERVO_PIN, HIGH);
  delayMicroseconds(pulseWidth);
  
  // Drive signal pin LOW for the remainder of the 20ms frame
  digitalWrite(SERVO_PIN, LOW);
  delayMicroseconds(20000 - pulseWidth);
  
  // Short delay to allow the servo physical transition time
  delay(15);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, a **Potentiometer**, and a **Servo** onto the canvas.
2. Wire the Potentiometer's center pin to **GP26 (ADC0)**.
3. Wire the Servo's VCC to **5V**, GND to **GND**, and Signal to **GPIO 13**.
4. Paste the code into the editor.
5. Click **Run**.
6. Rotate the virtual potentiometer slider and watch the servo arm rotate.

## Expected Output
Serial Monitor:
```
System Initialized.
```

## Expected Canvas Behavior
* The virtual servo widget's shaft rotates to match the angle set by the potentiometer.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `const int SERVO_PIN = 13;` | Identifies GPIO 13 as the servo control signal output. |
| `map(potVal, 0, 4095, 1000, 2000)` | Maps the 12-bit input (0-4095) to the standard servo pulse width range (1000 to 2000 microseconds). |
| `digitalWrite(SERVO_PIN, HIGH); delayMicroseconds(pulseWidth);` | Generates the active-high positioning pulse. |
| `digitalWrite(SERVO_PIN, LOW); delayMicroseconds(20000 - pulseWidth);` | Completes the rest of the 20ms frame (50 Hz signal period). |

## Hardware & Safety Concept: Servo Control Pulses and High Current Inductive Loads
* **Servo Control Pulses**: Hobby servos are controlled by sending a pulse width modulated (PWM) signal at 50 Hz (20ms period). The duration of the active-high pulse determines the angle: a 1.0ms pulse (1000 µs) sets the servo to 0°, a 1.5ms pulse (1500 µs) sets it to 90° (center), and a 2.0ms pulse (2000 µs) sets it to 180°.
* **High Current Inductive Loads**: Motors and servos are inductive loads. When starting up or stalled, they draw high currents (up to 1A) that can cause voltage dips (brownouts) on the microcontroller, resetting the board. For physical hardware projects involving multiple servos, always use an external power supply to power the servo rails, sharing only GND with the microcontroller.

## Try This! (Challenges)
1. **Slow Sweep Mode**: Modify the code to sweep the servo slowly and continuously between 0° and 180° automatically, ignoring the potentiometer.
2. **Range Bounds Tuning**: Some servos can rotate up to 180° but have physical stops that bind if driven too far. Readjust your mapping range (e.g. 800 to 2200 microseconds) to find the absolute physical limits of your servo without straining the motor.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo jitters or sweeps erratically | Poor ground connection or unstable power | Ensure the Brown/GND wire of the servo is securely connected to the same GND as the ARIES board |
| Servo does not move | Incorrect control pin or low voltage | Make sure the servo signal wire is plugged into GPIO 13 and that the servo is powered by 5V (not 3.3V) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [44 - Hall Effect magnetic sensor print](44-hall-effect-magnetic-sensor-print.md) (Previous project)
- [46 - Joysticks X-Axis read](46-joysticks-x-axis-read.md) (Next project)
