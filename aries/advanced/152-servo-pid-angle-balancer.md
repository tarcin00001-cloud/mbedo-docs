# 152 - Servo PID Angle Balancer

Maintain a horizontal platform level by driving a servo motor based on real-time pitch feedback from an MPU-6050 IMU sensor processed through a PID controller.

## Goal
Learn how to implement a PID balancing control loop on the VEGA ARIES v3 board using accelerometer-derived pitch calculations and a loop-free C++ state structure.

## What You Will Build
An automatic platform balancing system. An MPU-6050 sensor is mounted to a platform pivoted on a servo motor (GPIO 13). When the platform tilts, the MPU-6050 reads the raw acceleration values. The program computes the current pitch angle, applies a PID algorithm, and adjusts the servo position to counter the tilt, keeping the platform level.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MPU-6050 Accelerometer & Gyroscope | `mpu6050` | Yes | Yes |
| SG90 Servo Motor | `servo` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Servo Motor | Signal | GPIO 13 | Yellow | Servo PWM control line |
| Servo Motor | VCC | 5V | Red | Power supply |
| Servo Motor | GND | GND | Black | Ground reference |
| MPU-6050 | VCC | 3V3 | Red | Sensor operating voltage (3.3V) |
| MPU-6050 | GND | GND | Black | Ground reference |
| MPU-6050 | SDA | SDA0 (GP17) | Blue | I2C Data line |
| MPU-6050 | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** Pitch calculations require clean, noiseless sensor readings. Ensure the MPU-6050 has short I2C wires and is securely mounted to the platform driven by the servo arm.

## Code
```cpp
// Servo PID Angle Balancer - VEGA ARIES v3
#include <Wire.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <Servo.h>

Adafruit_MPU6050 mpu;
Servo myServo;

bool sensorOk = true;
unsigned long lastTime = 0;

// PID Tuning Constants
float Kp = 1.8;
float Ki = 0.5;
float Kd = 0.15;

float targetAngle = 0.0; // Horizontal goal (degrees)
float lastError = 0.0;
float integral = 0.0;

void setup() {
  Serial.begin(115200);
  Wire.begin();
  
  myServo.attach(13);
  myServo.write(90); // Center the servo

  if (!mpu.begin()) {
    Serial.println("Failed to find MPU-6050!");
    sensorOk = false;
  } else {
    mpu.setAccelerometerRange(MPU6050_RANGE_2_G);
    mpu.setFilterBandwidth(MPU6050_BAND_10_HZ);
    Serial.println("MPU-6050 initialized.");
  }
  
  lastTime = millis();
}

void loop() {
  if (!sensorOk) {
    delay(1000);
    return;
  }

  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);

  // Calculate Pitch Angle from accelerometer data: atan2(y, z)
  // Converting rad to degrees
  float pitch = atan2(a.acceleration.y, a.acceleration.z) * 180.0 / PI;

  unsigned long currentTime = millis();
  unsigned long elapsed = currentTime - lastTime;

  if (elapsed >= 50) { // PID calculation every 50ms
    float elapsedSec = (float)elapsed / 1000.0;
    lastTime = currentTime;

    float error = targetAngle - pitch;
    integral = integral + (error * elapsedSec);
    float derivative = (error - lastError) / elapsedSec;
    lastError = error;

    float output = (Kp * error) + (Ki * integral) + (Kd * derivative);
    
    // Convert PID output to servo command (neutral position at 90 degrees)
    int servoPos = 90 + (int)output;

    // Safety constraint to protect physical servo limits
    if (servoPos < 10) {
      servoPos = 10;
    }
    if (servoPos > 170) {
      servoPos = 170;
    }

    myServo.write(servoPos);

    Serial.print("Pitch: ");
    Serial.print(pitch, 1);
    Serial.print(" deg | Error: ");
    Serial.print(error, 1);
    Serial.print(" | Servo Pos: ");
    Serial.println(servoPos);
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, **MPU-6050**, and **Servo Motor** onto the canvas.
2. Connect the Servo Motor: **Signal** to **GPIO 13**, **VCC** to **5V**, and **GND** to **GND**.
3. Connect the MPU-6050: **VCC** to **3V3**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
4. Copy the project code and paste it into the editor.
5. Click **Run**.
6. Adjust the MPU-6050 tilt widget on the canvas. Observe how the servo position dynamically updates in response to counteract the tilt angle.

## Expected Output
Serial Monitor:
```
MPU-6050 initialized.
Pitch: 12.4 deg | Error: -12.4 | Servo Pos: 67
Pitch: 5.2 deg | Error: -5.2 | Servo Pos: 80
Pitch: 0.1 deg | Error: -0.1 | Servo Pos: 90
Pitch: -8.7 deg | Error: 8.7 | Servo Pos: 105
```

## Expected Canvas Behavior
* At startup, the servo positions itself at 90° (horizontal calibration point).
* Tilting the simulated MPU-6050 sensor causes the servo horn to rotate in the opposite direction to balance the platform back to 0° (flat position).

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `atan2(a.acceleration.y, a.acceleration.z)` | Computes the pitch angle based on relative gravity components on the Y and Z axes. |
| `elapsed >= 50` | Restricts balancing updates to a steady 20 Hz frequency to prevent servo chatter. |
| `error = targetAngle - pitch` | Calculates deviation from the horizontal target (0.0°). |
| `integral = integral + ...` | Accumulates error over time to eliminate steady-state offsets. |
| `servoPos = 90 + (int)output` | Offsets the control variable from the neutral 90° position. |
| `myServo.write(servoPos)` | Commands the servo to move to the updated balance position. |

## Hardware & Safety Concept
* **Anti-Chatter Filter**: Fast, continuous servo updates can cause excessive gear wear and high power consumption (chatter). Tuning the PID derivative term (`Kd`) and applying a mechanical band constraint prevents micro-oscillations.
* **Mechanical Stop Protection**: Constraining the servo output between 10° and 170° avoids running the servo into mechanical end stops, which could strip the gears or draw stall-level currents.

## Try This! (Challenges)
1. **Dynamic Target**: Connect a slide potentiometer to `ADC0` and use it to adjust the horizontal setpoint `targetAngle` between -30° and 30° on the fly.
2. **Aggressive Response**: Double the proportional value (`Kp = 3.6`) and notice the rapid but unstable balancing response.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The servo turns to its limit and stays there | Reversed feedback loop | Check pitch calculations. If the servo corrects in the same direction as the tilt, change the sign of the output: `servoPos = 90 - (int)output`. |
| MPU-6050 fails to initialize | Loose I2C wiring | Check SDA and SCL connections. Ensure you are using SDA0/SCL0 pin headers on ARIES. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [51 - Servo Motor 180 Degree Sweep](../intermediate/51-servo-motor-180-degree-sweep.md)
- [93 - MPU-6050 Accelerometer Serial Logs](../intermediate/93-mpu-6050-accelerometer-serial.md)
