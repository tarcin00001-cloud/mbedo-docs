# 57 - ESP32 Stepper Motor Speed Controller

Control the rotational speed of a 28BYJ-48 stepper motor by varying the step delay using a potentiometer, allowing real-time speed adjustment from slow to fast.

## Goal
Learn how step delay determines stepper motor speed, and how to map a potentiometer ADC reading to a variable step delay for live speed control.

## What You Will Build
A ULN2003 stepper driver with a 28BYJ-48 motor. A potentiometer on GPIO 34 controls the step delay (2–20 ms), which sets the motor speed. The motor runs continuously forward while the potentiometer adjusts speed in real time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| ULN2003 Stepper Driver Board | `stepper_driver` | Yes | Yes |
| 28BYJ-48 Stepper Motor | `stepper_motor` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| ULN2003 Board | IN1 | GPIO18 | Yellow | Coil phase 1 |
| ULN2003 Board | IN2 | GPIO19 | Green | Coil phase 2 |
| ULN2003 Board | IN3 | GPIO21 | Blue | Coil phase 3 |
| ULN2003 Board | IN4 | GPIO22 | Purple | Coil phase 4 |
| ULN2003 Board | VCC (+) | 5V (Vin) | Red | Motor coil supply |
| ULN2003 Board | GND (−) | GND | Black | Shared ground |
| Potentiometer | Left leg | 3V3 | Red | Reference voltage |
| Potentiometer | Right leg | GND | Black | Ground reference |
| Potentiometer | Wiper | GPIO34 | Yellow | Speed control input |

> **Wiring tip:** Reading the potentiometer inside the step loop (once per step) gives smooth real-time speed response without needing interrupts. Keep the potentiometer wiper wire short to avoid ADC noise affecting the step delay calculation.

## Code
```cpp
// Stepper Motor Speed Controller via potentiometer
const int IN1 = 18, IN2 = 19, IN3 = 21, IN4 = 22;
const int POT_PIN = 34;

const int STEPS[4][4] = {
  {1,0,0,0}, {0,1,0,0}, {0,0,1,0}, {0,0,0,1}
};

int stepIndex = 0;

void stepOnce() {
  digitalWrite(IN1, STEPS[stepIndex][0]);
  digitalWrite(IN2, STEPS[stepIndex][1]);
  digitalWrite(IN3, STEPS[stepIndex][2]);
  digitalWrite(IN4, STEPS[stepIndex][3]);
  stepIndex = (stepIndex + 1) % 4;
}

void setup() {
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT);
  digitalWrite(IN1,LOW); digitalWrite(IN2,LOW);
  digitalWrite(IN3,LOW); digitalWrite(IN4,LOW);

  Serial.begin(115200);
  Serial.println("Stepper Speed Controller ready — turn the potentiometer.");
}

void loop() {
  // Read potentiometer and map to step delay 2–20 ms
  int raw       = analogRead(POT_PIN);
  int stepDelay = map(raw, 0, 4095, 20, 2);   // Low ADC=slow, High ADC=fast

  stepOnce();
  delay(stepDelay);

  // Print speed info every 50 steps to avoid flooding Serial
  static int count = 0;
  if (++count % 50 == 0) {
    int rpm = 60000 / (stepDelay * 2048);   // Approx RPM
    Serial.print("Step delay: "); Serial.print(stepDelay);
    Serial.print(" ms  |  ~"); Serial.print(rpm);
    Serial.println(" RPM");
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer**, **ULN2003**, and **Stepper Motor** onto the canvas.
2. Connect Potentiometer **wiper** to **GPIO34**, IN1–IN4 to **GPIO18, 19, 21, 22**.
3. Paste the code and click **Run**.
4. Move the potentiometer slider — observe the stepper widget speed changing.

## Expected Output
Serial Monitor:
```
Stepper Speed Controller ready — turn the potentiometer.
Step delay: 18 ms  |  ~1 RPM
Step delay: 10 ms  |  ~2 RPM
Step delay: 4 ms   |  ~7 RPM
Step delay: 2 ms   |  ~14 RPM
```

## Expected Canvas Behavior
* Stepper widget rotates slowly when the slider is at the low end.
* Moving the slider higher increases rotation speed noticeably.
* Speed changes apply to the very next step — no lag.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `map(raw, 0, 4095, 20, 2)` | Inverted map: low ADC (pot left) = 20 ms delay (slow); high ADC = 2 ms delay (fast). |
| `stepOnce()` | Advances one step in the 4-step sequence and increments the index. |
| `stepIndex = (stepIndex + 1) % 4` | Cycles 0→1→2→3→0 continuously for forward rotation. |
| `count % 50 == 0` | Prints the status every 50 steps — approximately every 100 ms to 1 s depending on speed. |
| `60000 / (stepDelay * 2048)` | Approximate RPM formula: 60,000 ms/min ÷ (ms/step × 2048 steps/rev). |

## Hardware & Safety Concept: Step Rate and Motor Speed
Stepper motor speed is determined by **step rate** — how many steps per second are commanded. Speed (RPM) = step_rate × 60 ÷ steps_per_revolution. The 28BYJ-48's maximum reliable step rate is approximately 500 steps/second (2 ms/step) before torque drops significantly and steps are missed. Below 100 steps/second (~10 ms/step) the motor moves slowly but with maximum torque. This characteristic (maximum torque at low speed, zero torque above maximum speed) is fundamental to stepper motors and determines their use cases: slow, precise positioning applications like 3D printer axes, CNC plotters, and telescope mounts — not continuous high-speed rotation applications.

## Try This! (Challenges)
1. **Direction control**: Add a button on GPIO 4 that reverses the `stepIndex` count direction when pressed.
2. **Acceleration ramp**: Instead of instantly applying the potentiometer speed, gradually move the current delay toward the target to simulate smooth acceleration.
3. **RPM display**: Print RPM to a 16x2 I2C LCD (Project 58) instead of the Serial Monitor.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor skips and stalls at high speed | Step delay too short | Do not set below 2 ms; increase minimum mapping to 3 ms |
| Speed does not change with potentiometer | ADC not reading GPIO 34 | Verify wiper wire connection; check POT_PIN constant |
| Motor runs backward unexpectedly | Step sequence reversed | Confirm STEPS array order: row 0 = only IN1 HIGH |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [56 - ESP32 Stepper Motor Step Sequence](56-esp32-stepper-motor-step-sequence.md)
- [68 - ESP32 Stepper Motor Position Log](68-esp32-stepper-motor-position-log.md)
- [46 - ESP32 Potentiometer Servo Control](../beginner/46-esp32-potentiometer-servo-control.md)
