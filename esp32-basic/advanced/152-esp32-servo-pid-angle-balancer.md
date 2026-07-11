# 152 - ESP32 Servo PID Angle Balancer

Build a closed-loop stabilization platform that measures physical tilt angle (Roll) using an MPU-6050 sensor, and uses a PID loop to adjust a servo motor to keep the platform level (balanced at 0°).

## Goal
Learn how to compute angular errors, implement a PID control loop, map correction outputs to absolute servo positions, and build a stabilizer.

## What You Will Build
An MPU-6050 IMU is connected to I2C. A servo is on GPIO 13. The ESP32 measures the roll angle. If the platform is tilted, the code computes the error relative to 0° (flat) and runs a PID loop. The PID output adjusts the servo position (Base Angle: 90° plus/minus correction) to return the platform to level.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MPU-6050 6-DOF IMU Sensor | `mpu6050` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |
| External Power Supply (5V, 1A) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 Sensor | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| MPU-6050 Sensor | VCC / GND | 3V3 / GND | Red / Black | IMU Power |
| Servo Motor | Signal | GPIO13 | Orange | PWM servo control |
| Servo Motor | VCC / GND | 5V / GND | Red / Black | Servo Power |

> **Wiring tip:** Share the common ground between the servo and the ESP32. Power the servo from the 5V Vin rail to prevent voltage drops on the 3.3V line.

## Code
```cpp
// Servo PID Angle Balancer (MPU-6050 + Servo)
#include <Wire.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <ESP32Servo.h>
#include <math.h>

const int SERVO_PIN = 13;

Adafruit_MPU6050 mpu;
Servo balanceServo;

// PID Configuration Constants
const float KP = 1.5;
const float KI = 0.2;
const float KD = 0.05;

// Target balancing angle (0 degrees = flat level)
const float TARGET_ANGLE = 0.0; 

// Servo center position representing level
const int SERVO_CENTER = 90; 

float error = 0.0;
float lastError = 0.0;
float integral = 0.0;
float derivative = 0.0;

unsigned long lastTime = 0;

void setup() {
  Serial.begin(115200);
  
  if (!mpu.begin()) {
    Serial.println("MPU-6050 initialization failed!");
    while(1) {}
  }
  
  mpu.setAccelerometerRange(MPU6050_RANGE_4_G);
  
  balanceServo.attach(SERVO_PIN);
  balanceServo.write(SERVO_CENTER); // Start centered
  
  lastTime = millis();
  Serial.println("Angle Balancer online. Keep sensor flat.");
  delay(1000);
}

void loop() {
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);
  
  unsigned long now = millis();
  float dt = (now - lastTime) / 1000.0; // Time step in seconds
  
  if (dt <= 0.0) dt = 0.01; // Prevent division by zero
  lastTime = now;
  
  // Calculate Roll angle in degrees from gravity vector components
  // Roll (phi) = atan2(ay, az)
  float roll = atan2(a.acceleration.y, a.acceleration.z) * 180.0 / M_PI;
  
  // Calculate Error (Target - Current)
  error = TARGET_ANGLE - roll;
  
  // Calculate Integral with anti-windup constraint
  integral += error * dt;
  integral = constrain(integral, -30, 30);
  
  // Calculate Derivative
  derivative = (error - lastError) / dt;
  lastError = error;
  
  // Calculate PID Output
  float correction = (KP * error) + (KI * integral) + (KD * derivative);
  
  // Calculate target servo angle
  // Adding correction to center angle
  int targetServoAngle = SERVO_CENTER + (int)correction;
  
  // Constrain servo limits to prevent over-travel
  targetServoAngle = constrain(targetServoAngle, 30, 150);
  
  // Command servo
  balanceServo.write(targetServoAngle);
  
  // Log telemetry to Serial Plotter format
  Serial.print("Roll: "); Serial.print(roll, 1);
  Serial.print(" | Error: "); Serial.print(error, 1);
  Serial.print(" | Correction: "); Serial.print(correction, 1);
  Serial.print(" | Servo: "); Serial.println(targetServoAngle);
  
  delay(20); // 50Hz control loop frequency
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MPU-6050**, and **Servo** onto the canvas.
2. Wire MPU-6050 to **GPIO21/GPIO22** and Servo to **GPIO13**.
3. Paste the code and click **Run**.
4. Slide the MPU-6050 Y acceleration slider (tilting the sensor). Watch the servo rotate in the opposite direction to compensate.
5. Watch the servo return to 90° when the sensor is flat (Y = 0).

## Expected Output
Serial Monitor:
```
Angle Balancer online. Keep sensor flat.
Roll: 0.2 | Error: -0.2 | Correction: -0.3 | Servo: 89
Roll: 15.4 | Error: -15.4 | Correction: -23.1 | Servo: 66
Roll: -20.5 | Error: 20.5 | Correction: 30.7 | Servo: 120
```

## Expected Canvas Behavior
* Sliding the MPU-6050 Y-acceleration slider positive moves the servo widget arm counter-clockwise.
* Sliding the Y-acceleration slider negative moves the servo arm clockwise.
* The servo moves dynamically to counteract the tilt.

## Code Walkthrough
| Line | Math / Check |
| --- | --- |
| `atan2(ay, az) * 180.0 / M_PI` | Calculates the roll angle in degrees. |
| `error = TARGET_ANGLE - roll` | Computes the deviation from flat level. |
| `SERVO_CENTER + correction` | Adjusts the center point bias based on PID feedback. |

## Hardware & Safety Concept: PID Loop Tuning and Anti-Windup
PID controllers consist of three parameters:
1. **Proportional (P)**: Generates a correction proportional to the current error. If the error is large, the correction is large.
2. **Integral (I)**: Accumulates error over time to eliminate steady-state offsets (like gravity loading).
3. **Derivative (D)**: Measures the rate of change of the error, dampening the response to prevent overshoot and oscillations.
**Anti-windup** clamps the integral term to prevent it from accumulating to infinity if the servo hits a physical limit, avoiding sluggish response when the error reverses.

## Try This! (Challenges)
1. **Interactive PID Tuning**: Use the Serial Monitor to input tuning constants (e.g. `P1.2`, `I0.1`, `D0.02`) to test their effects.
2. **Tilt limit alarm**: Sound a buzzer (GPIO 4) if the roll angle exceeds 45 degrees.
3. **Complementary Filter Integration**: Combine accelerometer and gyroscope data to calculate cleaner tilt angles under vibration.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo rotates in the same direction as the tilt (amplifies error) | Correction sign inverted | Change the sign of the correction (e.g. `SERVO_CENTER - correction`) |
| Platform oscillates wildly back and forth | `KP` or `KD` too high | Reduce the proportional gain constant (e.g. set `KP = 0.8`) |
| Servo moves sluggishly | `KI` is too low | Increase the integral gain constant slightly or decrease the check interval |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [151 - ESP32 DC Motor PID Speed Controller](151-esp32-dc-motor-pid-speed-controller.md)
- [95 - ESP32 MPU-6050 Tilt Sensor Alarm](../intermediate/95-esp32-mpu6050-tilt-sensor-alarm.md)
- [155 - ESP32 OLED Complementary Filter Balancer](155-esp32-oled-complementary-filter-balancer.md)
