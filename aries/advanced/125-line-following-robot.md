# 125 - Line Following Robot

Build an autonomous line following robot using dual infrared (IR) line tracking sensors and DC motors driven by an L298N module.

## Goal
Understand the reflective principles of IR sensors, build closed-loop feedback path tracking control, and implement basic line-following logic.

## What You Will Build
An autonomous vehicle that tracks a black line painted on a white surface. Two IR reflective line sensors are mounted at the front. The ARIES board reads the digital outputs from these sensors to detect line deviations. When the robot drifts off the line, the code turns the appropriate motor off to pivot the robot back onto the path.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| 2x IR Line Tracking Sensors (TCRT5000) | `ir_sensor` | Yes | Yes |
| L298N Motor Driver Module | `l298n` | Yes | Yes |
| 2x DC Toy Motors | `dc_motor` | Yes | Yes |
| External 6V-12V Power Source | `power_supply` | Optional | Yes (battery pack) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Left IR Sensor | VCC | 3V3 | Red | Sensor operating voltage (3.3V) |
| Left IR Sensor | GND | GND | Black | Ground reference |
| Left IR Sensor | DO (Digital Out) | GPIO 9 | Orange | Left sensor signal line |
| Right IR Sensor | VCC | 3V3 | Red | Sensor operating voltage (3.3V) |
| Right IR Sensor | GND | GND | Black | Ground reference |
| Right IR Sensor | DO (Digital Out) | GPIO 8 | Yellow | Right sensor signal line |
| L298N Module | VCC / GND | 5V / GND | Red / Black | Power supply connections |
| L298N Module | IN1 / IN2 | GPIO 14 / 15 | Orange / Brown | Left motor controls |
| L298N Module | IN3 / IN4 | GPIO 13 / 12 | Green / Blue | Right motor controls |

> **Wiring tip:** Connect both IR sensor VCC lines to the ARIES 3.3V rail. The sensor digital output pins (DO) send a HIGH level when a dark line (low reflection) is detected, and a LOW level when a white surface (high reflection) is detected.

## Code
```cpp
// Line Following Robot - VEGA ARIES v3
const int LEFT_SENSOR = 9;   // Left Line Sensor on GPIO 9
const int RIGHT_SENSOR = 8;  // Right Line Sensor on GPIO 8

const int IN1 = 14;          // Left Motor Forward
const int IN2 = 15;          // Left Motor Backward
const int IN3 = 13;          // Right Motor Forward
const int IN4 = 12;          // Right Motor Backward

void setup() {
  // Configure sensor pins as inputs
  pinMode(LEFT_SENSOR, INPUT);
  pinMode(RIGHT_SENSOR, INPUT);

  // Configure motor driver control pins as outputs
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  // Initialize Serial Monitor for monitoring
  Serial.begin(9600);
  Serial.println("Line Following Robot Initialized.");
}

void loop() {
  // Read digital outputs of the line sensors
  // HIGH indicates black line, LOW indicates white surface
  int leftState = digitalRead(LEFT_SENSOR);
  int rightState = digitalRead(RIGHT_SENSOR);

  if (leftState == LOW && rightState == LOW) {
    // Case 1: Both sensors on white surface -> Go straight forward
    Serial.println("State: ON TRACK - Moving Forward");
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, HIGH);
    digitalWrite(IN4, LOW);
  } 
  else if (leftState == HIGH && rightState == LOW) {
    // Case 2: Left sensor is on black line -> Pivot Left (turn off Left motor)
    Serial.println("State: DRIFTED RIGHT - Correcting Left");
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, HIGH);
    digitalWrite(IN4, LOW);
  } 
  else if (leftState == LOW && rightState == HIGH) {
    // Case 3: Right sensor is on black line -> Pivot Right (turn off Right motor)
    Serial.println("State: DRIFTED LEFT - Correcting Right");
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, LOW);
  } 
  else {
    // Case 4: Both sensors on black line -> Stop (junction or end of line)
    Serial.println("State: JUNCTION/STOP - Stopping");
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, LOW);
  }

  delay(20); // Small delay to prevent rapid toggle noise
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **L298N Motor Driver**, two **DC Toy Motors**, and two **IR Line Sensors** onto the canvas.
2. Wire the L298N module VCC/GND and outputs to the DC motors.
3. Connect L298N inputs: **IN1** to **GPIO 14**, **IN2** to **GPIO 15**, **IN3** to **GPIO 13**, and **IN4** to **GPIO 12**.
4. Connect the Left Sensor DO to **GPIO 9** and the Right Sensor DO to **GPIO 8**.
5. Paste the C++ code into the editor.
6. Select **Interpreted Mode** in the simulation dropdown.
7. Click **Run**. Toggle the virtual IR sensor states to test the motor response.

## Expected Output
Serial Monitor:
```
Line Following Robot Initialized.
State: ON TRACK - Moving Forward
State: DRIFTED RIGHT - Correcting Left
State: ON TRACK - Moving Forward
State: DRIFTED LEFT - Correcting Right
State: JUNCTION/STOP - Stopping
```

## Expected Canvas Behavior
* If both simulated sensors are LOW, both motors spin forward.
* Toggling the left sensor widget to HIGH stops the left motor, keeping the right motor spinning.
* Toggling the right sensor widget to HIGH stops the right motor, keeping the left motor spinning.
* Toggling both sensors to HIGH stops both motors.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(LEFT_SENSOR, INPUT)` | Configures the left IR sensor signal line as a digital input. |
| `digitalRead(LEFT_SENSOR)` | Reads the reflection state (HIGH = black line, LOW = white surface). |
| `leftState == LOW && rightState == LOW` | Matches the straight-forward condition (centered on the path). |
| `digitalWrite(IN1, LOW)` | Disables Left Motor, causing the robot to swing left under the force of the active Right Motor. |

## Hardware & Safety Concept
* **Infrared Sensor Principles**: Line sensors emit infrared light from an IR LED and measure the reflected signal using a phototransistor. Dark materials (like a black tape track) absorb IR light, yielding low reflection (HIGH output from the sensor comparator). Light surfaces reflect IR light back, resulting in high reflection (LOW output).
* **Sensor Calibrations**: Ambient sunlight or different floor textures affect reflection thresholds. Real sensors feature a small potentiometer. Adjust it on site so that the onboard indicator LED turns on when placed over the black tape and turns off when placed over the white surface.

## Try This! (Challenges)
1. **Active Spin Correction**: Modify the turn cases to actively spin the wheels in opposite directions (e.g. during a left correction, run Left motor backward and Right motor forward) to execute tighter turns on sharp tracks.
2. **Speed Scaling Line Follower**: Incorporate speed scaling (similar to Project 123). Reduce the base speed of both motors as they correct to prevent overshoot on tight curves.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The robot spins continuously in circles | Motor wiring or sensor logic is reversed | Swap the left and right sensor pins in the code or reverse the motor wiring to correct rotation. |
| Robot fails to turn and runs off track | Low sensor sensitivity or slow execution | Adjust the physical potentiometer on the IR sensor board. Ensure the sampling delay is small (e.g. 10-20 ms). |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [37 - IR Obstacle Sensor Digital Read](../beginner/37-ir-obstacle-sensor-digital-read.md)
- [126 - Line Following Robot with Junction Stops](126-line-following-robot-with-junction-stops.md)
