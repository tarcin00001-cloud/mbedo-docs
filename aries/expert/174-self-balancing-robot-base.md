# 174 - Self-Balancing Robot Base

Design a self-balancing two-wheeled robot platform using an MPU-6050 Inertial Measurement Unit (IMU), L298N motor driver, and a PID controller.

## Goal
Learn how to interface an I2C-based 6-axis IMU, process accelerometer and gyroscope readings into pitch angle using a complementary filter, implement a closed-loop PID control algorithm using state variables, and dynamically steer motor PWM.

## What You Will Build
A two-wheeled balancing platform that maintains its upright position. The MPU-6050 reads raw acceleration and angular velocity. The ARIES board calculates the robot's tilt angle (pitch). The PID loop compares this pitch to a calibrated upright angle (setpoint) and adjusts the speed and direction of the DC motors in real time to correct the tilt.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MPU-6050 IMU | `mpu6050` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| 2x DC Motors | `dc_motor` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Warning LED | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 | VCC | 3V3 | Red | Power input (3.3V) |
| MPU-6050 | GND | GND | Black | Ground |
| MPU-6050 | SDA | GPIO 17 | Green | I2C Data (SDA0) |
| MPU-6050 | SCL | GPIO 16 | Yellow | I2C Clock (SCL0) |
| L298N Driver | ENA | GPIO 9 | Orange | Left motor PWM control |
| L298N Driver | IN1 | GPIO 7 | Yellow | Left motor direction 1 |
| L298N Driver | IN2 | GPIO 8 | Green | Left motor direction 2 |
| L298N Driver | ENB | GPIO 10 | Blue | Right motor PWM control |
| L298N Driver | IN3 | GPIO 3 | Purple | Right motor direction 1 |
| L298N Driver | IN4 | GPIO 4 | Gray | Right motor direction 2 |
| L298N Driver | VCC | 5V | Red | Motor power |
| L298N Driver | GND | GND | Black | Common Ground |
| Potentiometer | Wiper | ADC0 | Blue | PID Tuning Setpoint (GP26) |
| Potentiometer | VCC | 3V3 | Red | Reference Voltage |
| Potentiometer | GND | GND | Black | Ground |
| Active Buzzer | + | GPIO 14 | Gray | Alarm pin |
| Active Buzzer | - | GND | Black | Ground |
| Warning LED | Anode | GPIO 15 | Red | Tilt warning indicator |
| Warning LED | Cathode | GND | Black | Ground |

> **Wiring tip:** The MPU-6050 communicates via I2C0 (GPIO 16/17). Ensure pull-up resistors are included on the SDA and SCL lines if your breakout module does not include them.

## Code
```cpp
// Self-Balancing Robot Base - VEGA ARIES v3
#include <Wire.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>

const int ENA = 9;
const int IN1 = 7;
const int IN2 = 8;
const int ENB = 10;
const int IN3 = 3;
const int IN4 = 4;
const int POT_PIN = ADC0;
const int BUZZER = 14;
const int LED_PIN = 15;

Adafruit_MPU6050 mpu;

// PID Variables
float Kp = 25.0;
float Ki = 120.0;
float Kd = 1.2;

float targetAngle = 0.0; // Dynamic setpoint from Potentiometer
float currentAngle = 0.0;
float error = 0.0;
float lastError = 0.0;
float integral = 0.0;
float derivative = 0.0;
float pidOutput = 0.0;

// Filter Variables
float gyroAngle = 0.0;
float accAngle = 0.0;
unsigned long lastTime = 0;
float dt = 0.01; // 10ms loop time

void setup() {
  Serial.begin(115200);
  Wire.begin();
  
  pinMode(ENA, OUTPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(ENB, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  pinMode(BUZZER, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  
  if (!mpu.begin()) {
    Serial.println("Failed to find MPU-6050 chip");
    digitalWrite(LED_PIN, HIGH);
    delay(5000); // Fatal pause
  }
  
  mpu.setAccelerometerRange(MPU6050_RANGE_2_G);
  mpu.setGyroRange(MPU6050_RANGE_250_DEG);
  mpu.setFilterBandwidth(MPU6050_BAND_21_HZ);
  
  lastTime = millis();
}

void loop() {
  unsigned long currentTime = millis();
  unsigned long elapsed = currentTime - lastTime;
  
  // Run PID and filter logic every 10ms
  if (elapsed >= 10) {
    dt = (float)elapsed / 1000.0;
    lastTime = currentTime;
    
    // Read MPU-6050 values
    sensors_event_t a, g, temp;
    mpu.getEvent(&a, &g, &temp);
    
    // Calculate angle from accelerometer
    // Pitch angle: atan2 of Y and Z axes
    accAngle = atan2(a.acceleration.y, a.acceleration.z) * 180.0 / PI;
    
    // Angular rate from gyroscope (X axis is the tilt axis)
    float gyroRate = g.gyro.x * 180.0 / PI; // convert rad/s to deg/s
    
    // Complementary Filter: 98% Gyro (high speed), 2% Accel (low drift)
    currentAngle = 0.98 * (currentAngle + gyroRate * dt) + 0.02 * accAngle;
    
    // Read dynamic center calibration from Potentiometer (-5 to +5 degrees)
    float potVal = (float)analogRead(POT_PIN);
    targetAngle = map(potVal, 0, 1023, -50, 50) / 10.0;
    
    // PID Calculations
    error = targetAngle - currentAngle;
    integral = integral + (error * dt);
    
    // Constrain integral to prevent integrator windup
    if (integral > 100.0) integral = 100.0;
    if (integral < -100.0) integral = -100.0;
    
    derivative = (error - lastError) / dt;
    lastError = error;
    
    pidOutput = (Kp * error) + (Ki * integral) + (Kd * derivative);
    
    // Safety check: if robot falls over past 45 degrees, shut off motors
    if (currentAngle > 45.0 || currentAngle < -45.0) {
      analogWrite(ENA, 0);
      analogWrite(ENB, 0);
      digitalWrite(BUZZER, HIGH);
      digitalWrite(LED_PIN, HIGH);
      integral = 0.0; // reset integrator
    } 
    else {
      digitalWrite(BUZZER, LOW);
      digitalWrite(LED_PIN, LOW);
      
      // Determine direction and PWM duty cycle
      int motorSpeed = abs((int)pidOutput);
      if (motorSpeed > 255) motorSpeed = 255;
      
      if (pidOutput > 0) {
        // Drive Forward to catch balance
        digitalWrite(IN1, HIGH);
        digitalWrite(IN2, LOW);
        digitalWrite(IN3, HIGH);
        digitalWrite(IN4, LOW);
      } else {
        // Drive Backward to catch balance
        digitalWrite(IN1, LOW);
        digitalWrite(IN2, HIGH);
        digitalWrite(IN3, LOW);
        digitalWrite(IN4, HIGH);
      }
      
      analogWrite(ENA, motorSpeed);
      analogWrite(ENB, motorSpeed);
    }
    
    // Serial debugging output
    Serial.print("Angle: ");
    Serial.print(currentAngle);
    Serial.print(" | Target: ");
    Serial.print(targetAngle);
    Serial.print(" | PID: ");
    Serial.println(pidOutput);
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **MPU-6050**, **L298N Driver**, **two DC Motors**, **Potentiometer**, **Buzzer**, and **LED** onto the canvas.
2. Wire up the components. Connect SDA/SCL to the analog header SDA0/SCL0.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the dropdown.
5. Click **Run**.
6. Alter the MPU-6050 widget pitch angle slider to simulate physical tilt. The motors should spin forward or backward to correct the angle.

## Expected Output
Serial Monitor:
```
Angle: 2.30 | Target: 0.00 | PID: 57.50
Angle: 0.15 | Target: 0.00 | PID: 3.75
Angle: -1.20 | Target: 0.00 | PID: -30.00
Angle: 48.00 | Target: 0.00 | PID: 0.00 (Safety Shutdown!)
```

## Expected Canvas Behavior
* As the pitch slider of the MPU-6050 is moved in the positive direction (tilt forward), the DC motors spin in the forward direction.
* When moved in the negative direction, the DC motors reverse direction.
* The intensity of motor rotation (PWM speed) is proportional to the size and rate of the angle change.
* If the angle slider exceeds 45 degrees, the motors stop completely, the buzzer sounds continuously, and the warning LED illuminates.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `mpu.begin()` | Queries the MPU-6050 over the I2C0 bus and initializes its internal registers. |
| `currentAngle = ...` | Combines integration of gyroscope rate with accelerometer tilt calculation to filter noise. |
| `error = targetAngle - ...` | Evaluates deviation from the desired upright setpoint angle. |
| `integral = integral + ...` | Accumulates systemic offset over time to account for unequal weight distribution. |
| `pidOutput = ...` | Computes proportional, integral, and derivative terms into one motor torque value. |
| `currentAngle > 45.0 ...` | Safety threshold check prevents runaway motor spin when the robot falls completely. |

## Hardware & Safety Concept
* **Integrator Windup Prevention**: If the robot is physically held down while leaning, the integral term will accumulate to infinity. Constraining the integral limit ensures the PID controller can recover instantly.
* **Gyroscope Calibration Offset**: Real gyroscopes suffer from zero-rate drift. In a physical implementation, we would subtract the calibration offset determined at boot time when the robot is held perfectly still.

## Try This! (Challenges)
1. **Adaptive Gain**: Modify the code so that `Kp` increases dynamically as the tilt angle increases, providing stiffer correction when farther from upright.
2. **Tilt Fall Recovery**: Implement a state where if the robot tilts beyond 45 degrees, it waits for the potentiometer value to be set to zero before allowing the motors to spin again.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot runs away and falls | Motor directions reversed | Swap the polarity wires of both motors or swap the high/low states of IN1/IN2 and IN3/IN4 in the code. |
| Robot oscillates violently | Kp or Kd too high | Reduce Kp by 30% or increase Kd slightly to dampen the oscillation. |
| MPU-6050 fails to start | SCL/SDA swapped or address wrong | Check SDA0/SCL0 wiring (pins 17/16). Ensure Vin is connected to 3.3V. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [152 - Servo PID Angle Balancer](../advanced/152-servo-pid-angle-balancer.md)
- [157 - MPU-6050 3D Orientation](../advanced/157-mpu-6050-3d-orientation.md)
- [185 - Drone Flight Controller Base](185-drone-flight-controller-base.md)
