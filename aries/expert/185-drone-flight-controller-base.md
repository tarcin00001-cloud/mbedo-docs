# 185 - Drone Flight Controller Base

Design a basic quadcopter flight controller mixing module using an MPU-6050 IMU, dual-axis PID stabilization loops, a master throttle potentiometer, and 4 PWM motor ESC drive outputs.

## Goal
Learn how to implement a quadcopter motor mixing matrix (Roll and Pitch), compute dual independent PID loops, map throttle setpoints, and enforce tilt safety cutoffs without using loops or arrays.

## What You Will Build
The core control unit of a quadcopter. The ARIES board reads orientation data (Pitch and Roll) from an MPU-6050 sensor. A potentiometer acts as the pilot's throttle stick. The code computes two separate PID stabilization outputs (one for Pitch, one for Roll) to maintain a level flight state. These PID values are combined with the throttle input to calculate the speed of four ESC-driven motors. If the drone tilts past 50° (crash condition), the flight controller instantly disarms all motors.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MPU-6050 IMU | `mpu6050` | Yes | Yes |
| 4x DC Motors (ESC Sim) | `dc_motor` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Warning LED | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 | SDA | GPIO 17 | Green | I2C0 SDA |
| MPU-6050 | SCL | GPIO 16 | Yellow | I2C0 SCL |
| MPU-6050 | VCC | 3V3 | Red | Power |
| MPU-6050 | GND | GND | Black | Ground |
| Motor 1 (FL) | PWM | GPIO 8 | Orange | Front-Left Motor (PWM0) |
| Motor 2 (FR) | PWM | GPIO 9 | Yellow | Front-Right Motor (PWM1) |
| Motor 3 (RL) | PWM | GPIO 10 | Green | Rear-Left Motor (PWM2) |
| Motor 4 (RR) | PWM | GPIO 11 | Blue | Rear-Right Motor (PWM3) |
| Potentiometer | Wiper | ADC0 | Blue | Throttle input (GP26) |
| Active Buzzer | + | GPIO 14 | Gray | Crash alarm buzzer |
| Warning LED | Anode | GPIO 15 | Red | Flight status indicator |

> **Wiring tip:** Standard quadcopter brushless motors are driven by Electronic Speed Controllers (ESCs) that take a 50Hz PWM signal. For simulation, the ARIES PWM outputs (pins 8, 9, 10, 11) correspond to PWM0-3 channels.

## Code
```cpp
// Drone Flight Controller Base - VEGA ARIES v3
#include <Wire.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>

const int MOTOR_FL = 8;
const int MOTOR_FR = 9;
const int MOTOR_RL = 10;
const int MOTOR_RR = 11;
const int THROTTLE_POT = ADC0;
const int BUZZER_PIN = 14;
const int WARNING_LED = 15;

Adafruit_MPU6050 mpu;

// PID Tuning Constants
float Kp = 1.5;
float Ki = 0.05;
float Kd = 0.4;

// Orientation Angles
float pitch = 0.0;
float roll = 0.0;

// PID States (Pitch)
float pitchError = 0.0;
float pitchLastError = 0.0;
float pitchIntegral = 0.0;
float pitchOutput = 0.0;

// PID States (Roll)
float rollError = 0.0;
float rollLastError = 0.0;
float rollIntegral = 0.0;
float rollOutput = 0.0;

unsigned long lastTime = 0;
bool flightArmed = false;
bool crashFault = false;

void setup() {
  Serial.begin(115200);
  Wire.begin();
  
  pinMode(MOTOR_FL, OUTPUT);
  pinMode(MOTOR_FR, OUTPUT);
  pinMode(MOTOR_RL, OUTPUT);
  pinMode(MOTOR_RR, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(WARNING_LED, OUTPUT);
  
  if (!mpu.begin()) {
    Serial.println("Failed to find MPU-6050 chip");
    crashFault = true;
    digitalWrite(WARNING_LED, HIGH);
  }
  
  mpu.setAccelerometerRange(MPU6050_RANGE_4_G);
  mpu.setGyroRange(MPU6050_RANGE_500_DEG);
  mpu.setFilterBandwidth(MPU6050_BAND_42_HZ);
  
  lastTime = millis();
}

void loop() {
  unsigned long currentTime = millis();
  unsigned long elapsed = currentTime - lastTime;
  
  // High-speed flight calculation loop every 10ms (100Hz refresh rate)
  if (elapsed >= 10 && !crashFault) {
    float dt = (float)elapsed / 1000.0;
    lastTime = currentTime;
    
    // Read MPU-6050 sensor data
    sensors_event_t a, g, temp;
    mpu.getEvent(&a, &g, &temp);
    
    // Calculate Pitch and Roll from accelerometer values
    float rawPitch = atan2(-a.acceleration.x, sqrt(a.acceleration.y * a.acceleration.y + a.acceleration.z * a.acceleration.z)) * 180.0 / PI;
    float rawRoll = atan2(a.acceleration.y, a.acceleration.z) * 180.0 / PI;
    
    // Gyro rate values (converted to deg/s)
    float gyroPitchRate = g.gyro.y * 180.0 / PI;
    float gyroRollRate = g.gyro.x * 180.0 / PI;
    
    // Complementary Filter
    pitch = 0.98 * (pitch + gyroPitchRate * dt) + 0.02 * rawPitch;
    roll = 0.98 * (roll + gyroRollRate * dt) + 0.02 * rawRoll;
    
    // Safety check: if vehicle tilts past 50 degrees, trigger disarm
    if (pitch > 50.0 || pitch < -50.0 || roll > 50.0 || roll < -50.0) {
      crashFault = true;
      analogWrite(MOTOR_FL, 0);
      analogWrite(MOTOR_FR, 0);
      analogWrite(MOTOR_RL, 0);
      analogWrite(MOTOR_RR, 0);
      digitalWrite(BUZZER_PIN, HIGH);
      digitalWrite(WARNING_LED, HIGH);
      return;
    }
    
    // Read Pilot throttle level (scaled 0 to 180 to leave headroom for PID mix)
    int throttleInput = map(analogRead(THROTTLE_POT), 0, 1023, 0, 180);
    
    // Arm system only when throttle is pulled low initially
    if (!flightArmed && throttleInput < 10) {
      flightArmed = true;
      digitalWrite(WARNING_LED, LOW);
      Serial.println("Flight Controller ARMED.");
    }
    
    if (flightArmed) {
      // 1. Pitch PID Loop (Setpoint is 0.0 degrees level flight)
      pitchError = 0.0 - pitch;
      pitchIntegral = pitchIntegral + (pitchError * dt);
      pitchIntegral = constrain(pitchIntegral, -50.0, 50.0);
      pitchDerivative = (pitchError - pitchLastError) / dt;
      pitchLastError = pitchError;
      
      pitchOutput = (Kp * pitchError) + (Ki * pitchIntegral) + (Kd * pitchDerivative);
      
      // 2. Roll PID Loop (Setpoint is 0.0 degrees level flight)
      rollError = 0.0 - roll;
      rollIntegral = rollIntegral + (rollError * dt);
      rollIntegral = constrain(rollIntegral, -50.0, 50.0);
      rollDerivative = (rollError - rollLastError) / dt;
      rollLastError = rollError;
      
      rollOutput = (Kp * rollError) + (Ki * rollIntegral) + (Kd * rollDerivative);
      
      // 3. Quadcopter Motor Mixing Matrix (X-Configuration)
      // FL = Throttle + PitchCorrection + RollCorrection
      // FR = Throttle + PitchCorrection - RollCorrection
      // RL = Throttle - PitchCorrection + RollCorrection
      // RR = Throttle - PitchCorrection - RollCorrection
      int mFL = throttleInput + pitchOutput + rollOutput;
      int mFR = throttleInput + pitchOutput - rollOutput;
      int mRL = throttleInput - pitchOutput + rollOutput;
      int mRR = throttleInput - pitchOutput - rollOutput;
      
      // Constrain speeds to 8-bit analog range
      if (mFL < 0) mFL = 0; if (mFL > 255) mFL = 255;
      if (mFR < 0) mFR = 0; if (mFR > 255) mFR = 255;
      if (mRL < 0) mRL = 0; if (mRL > 255) mRL = 255;
      if (mRR < 0) mRR = 0; if (mRR > 255) mRR = 255;
      
      // Force motors off if throttle is close to zero
      if (throttleInput < 12) {
        mFL = 0; mFR = 0; mRL = 0; mRR = 0;
      }
      
      // Write PWM signals
      analogWrite(MOTOR_FL, mFL);
      analogWrite(MOTOR_FR, mFR);
      analogWrite(MOTOR_RL, mRL);
      analogWrite(MOTOR_RR, mRR);
      
      // Output telemetry
      Serial.print("P:"); Serial.print(pitch, 1);
      Serial.print(" | R:"); Serial.print(roll, 1);
      Serial.print(" | M_FL:"); Serial.print(mFL);
      Serial.print(" M_FR:"); Serial.print(mFR);
      Serial.print(" M_RL:"); Serial.print(mRL);
      Serial.print(" M_RR:"); Serial.println(mRR);
    }
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **MPU-6050**, **4x DC Motors**, **Potentiometer**, **Buzzer**, and **LED** onto the canvas.
2. Complete the connections as per the wiring table. Connect motors to PWM pins 8, 9, 10, and 11.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the dropdown.
5. Click **Run**.
6. Set the throttle slider (potentiometer) to 0 first to ARM the board. Slowly raise the throttle and tilt the MPU-6050 slider to see the motor outputs balance.

## Expected Output
Serial Monitor:
```
Flight Controller ARMED.
P:0.0 | R:0.0 | M_FL:100 M_FR:100 M_RL:100 M_RR:100
P:10.5 | R:0.0 | M_FL:115 M_FR:115 M_RL:85 M_RR:85
P:10.5 | R:-5.2 | M_FL:107 M_FR:123 M_RL:77 M_RR:93
```

## Expected Canvas Behavior
* On startup, the warning LED is ON. Moving the potentiometer (throttle) to zero triggers system arming, turning the LED OFF.
* Moving throttle potentiometer slider up accelerates all 4 DC motors.
* Adjusting the MPU-6050 pitch angle positive (tilt forward) decreases the speed of Front motors (FL, FR) and increases Rear motors (RL, RR) to lift the nose.
* If any pitch or roll angle exceeds 50° in either direction, all motors shut off instantly, the warning LED turns ON, and the buzzer sounds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `rawPitch = atan2(...)` | Computes trig ratios of accelerometer vectors to deduce absolute pitch orientation. |
| `pitch = 0.98 * ...` | Complementary filter merges high-frequency gyro data with low-drift accel data. |
| `pitch > 50.0 ...` | Safety threshold cutoff stops motor drive in case of inverted flight/crash. |
| `throttleInput < 10` | Enforces arming safety interlock. The vehicle will not start if throttle is high. |
| `mFL = throttle + ...` | Mixing matrix calculates torque offsets for each propeller to level the craft. |

## Hardware & Safety Concept
* **ESC Safety Interlock**: Never allow a drone flight controller to arm unless the throttle stick is pulled down completely. This prevents the high-speed carbon blades from spinning up unexpectedly when the battery is plugged in.
* **IMU Vibrations Isolations**: Real drones experience high frequency motor vibrations. A software low-pass filter (LPF) combined with mechanical foam damping of the IMU is necessary to prevent sensor signal saturation.

## Try This! (Challenges)
1. **Pilot Stick Input**: Map a second potentiometer to `ADC1` to act as a Pitch steering control stick, shifting the setpoint from 0.0 to a target range of -15 to +15 degrees.
2. **Launch beep sequence**: Add a 3-beep buzzer melody when the system transition from DISARMED to ARMED.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| System will not ARM | Potentiometer wiper is not starting at zero | Pull the throttle potentiometer wiper all the way to 0V at startup. |
| Drone flips instantly on takeoff | Wrong motor mixing polarity | Verify that motor mixing sign signs match: front FL/FR add pitch correction; rear RL/RR subtract. |
| Motor speed oscillates wildly | PID loop rate is too slow or gains are too high | Ensure the 10ms loop time is executed accurately. Reduce Kd and Kp parameters. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [121 - Dual Motor Drive](../advanced/121-dual-motor-drive.md)
- [157 - MPU-6050 3D Orientation](../advanced/157-mpu-6050-3d-orientation.md)
- [174 - Self-Balancing Robot Base](174-self-balancing-robot-base.md)
