# 157 - MPU-6050 3D Orientation Visualizer

Calculate and stream three-dimensional Roll, Pitch, and Yaw orientation angles from an MPU-6050 IMU sensor over Serial telemetry.

## Goal
Learn how to use mathematical sensor fusion equations in C++ to determine 3D orientation, integrating gyroscope rate data over time for relative yaw tracking without loops.

## What You Will Build
A 3D orientation tracking node. The MPU-6050 reads accelerometer gravity vectors and gyroscope rotational velocities. The code calculates Roll and Pitch (absolute angles relative to gravity) and integrates the Gyro Z-axis to estimate relative Yaw. It outputs a formatted data packet over Serial (e.g., `Roll,Pitch,Yaw`) that can be parsed by 3D visualization utilities.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MPU-6050 Accelerometer & Gyroscope | `mpu6050` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 | VCC | 3V3 | Red | Sensor operating voltage (3.3V) |
| MPU-6050 | GND | GND | Black | Ground reference |
| MPU-6050 | SDA | SDA0 (GP17) | Blue | I2C Data line |
| MPU-6050 | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** Tie VCC to the 3.3V pin on the ARIES board. The I2C pins SDA and SCL must match I2C0 pins GP17 and GP16 on the VEGA ARIES v3 board layout.

## Code
```cpp
// MPU-6050 3D Orientation Visualizer - VEGA ARIES v3
#include <Wire.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>

Adafruit_MPU6050 mpu;
bool mpuOk = true;

// Orientation values
float roll = 0.0;
float pitch = 0.0;
float yaw = 0.0;

unsigned long lastTime = 0;
const float ALPHA = 0.96; // Complementary filter constant

void setup() {
  Serial.begin(115200);
  Wire.begin();

  if (!mpu.begin()) {
    Serial.println("Failed to find MPU-6050!");
    mpuOk = false;
  } else {
    mpu.setAccelerometerRange(MPU6050_RANGE_4_G);
    mpu.setGyroRange(MPU6050_RANGE_500_DEG);
    mpu.setFilterBandwidth(MPU6050_BAND_21_HZ);
    Serial.println("MPU-6050 Initialized successfully.");
  }
  
  lastTime = millis();
}

void loop() {
  if (!mpuOk) {
    delay(1000);
    return;
  }

  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);

  unsigned long currentTime = millis();
  unsigned long elapsed = currentTime - lastTime;

  // Stream data at 25Hz (every 40ms)
  if (elapsed >= 40) {
    float dt = (float)elapsed / 1000.0;
    lastTime = currentTime;

    // 1. Calculate absolute Roll and Pitch from accelerometer data
    // Roll: tilt about the X-axis
    float rollAccel = atan2(a.acceleration.y, sqrt(a.acceleration.x * a.acceleration.x + a.acceleration.z * a.acceleration.z)) * 180.0 / PI;
    // Pitch: tilt about the Y-axis
    float pitchAccel = atan2(-a.acceleration.x, sqrt(a.acceleration.y * a.acceleration.y + a.acceleration.z * a.acceleration.z)) * 180.0 / PI;

    // 2. Convert gyroscope rate data from rad/s to deg/s
    float gyroRateX = g.gyro.x * 180.0 / PI;
    float gyroRateY = g.gyro.y * 180.0 / PI;
    float gyroRateZ = g.gyro.z * 180.0 / PI;

    // 3. Apply Complementary Filter to fuse Roll and Pitch
    roll = ALPHA * (roll + gyroRateX * dt) + (1.0 - ALPHA) * rollAccel;
    pitch = ALPHA * (pitch + gyroRateY * dt) + (1.0 - ALPHA) * pitchAccel;

    // 4. Integrate Gyro Z rate directly to estimate relative Yaw (no accelerometer reference)
    // Add small deadband filtering to reduce gyroscope resting drift
    if (gyroRateZ > 0.5 || gyroRateZ < -0.5) {
      yaw = yaw + (gyroRateZ * dt);
    }

    // Keep Yaw within -180 to +180 degrees range
    if (yaw > 180.0) {
      yaw = yaw - 360.0;
    }
    if (yaw < -180.0) {
      yaw = yaw + 360.0;
    }

    // Output formatted visualizer telemetry payload
    Serial.print("Orientation: ");
    Serial.print(roll, 1);
    Serial.print(",");
    Serial.print(pitch, 1);
    Serial.print(",");
    Serial.println(yaw, 1);
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** and **MPU-6050** components onto the canvas.
2. Wire the MPU-6050: **VCC** to **3V3**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
3. Copy the project code and paste it into the editor.
4. Click **Run**.
5. Select the MPU-6050 widget and tilt it on the pitch or roll axis. Observe the changing values in the Serial Monitor.

## Expected Output
Serial Monitor:
```
MPU-6050 Initialized successfully.
Orientation: 0.5,1.2,0.0
Orientation: 12.4,-3.8,1.2
Orientation: 45.1,10.2,4.5
```

## Expected Canvas Behavior
* On startup, the serial prints confirmation of the MPU-6050.
* Rotating the MPU-6050 widget changes the Roll and Pitch output angles proportionally.
* Spinning the widget continuously updates the Yaw orientation angle.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `sqrt(a.acceleration.x * a.acceleration.x ...)` | Computes the magnitude of the horizontal vector to normalize pitch calculations. |
| `gyroRateX = g.gyro.x * 180.0 / PI` | Scales the raw angular rate of change from radians to degrees. |
| `roll = ALPHA * ...` | Reduces vibration noise on Roll using the complementary filter. |
| `yaw = yaw + (gyroRateZ * dt)` | Integrates Z-axis speed over time to track absolute yaw rotation. |
| `yaw > 180.0` | Wraps the yaw value around to keep it between -180° and 180°. |

## Hardware & Safety Concept
* **Yaw Drift Phenomenon**: Unlike Roll and Pitch, which are referenced to the absolute vector of gravity, Yaw has no absolute reference on a standard IMU without a magnetometer. Direct integration causes small gyro offsets to accumulate into drift over time.
* **Vibration Dampening**: In high-vibration environments (like quadcopters), high-frequency mechanical noise saturates accelerometer inputs. The code uses `mpu.setFilterBandwidth(MPU6050_BAND_21_HZ)` to filter out high-frequency mechanical vibrations in hardware.

## Try This! (Challenges)
1. **Calibration Mode**: In setup, read 100 samples of the gyroscope, calculate the average offset, and subtract it from `gyroRateZ` to minimize yaw drift.
2. **Indicator LEDs**: Turn ON the onboard Green LED (`LED_G` / GPIO 24) when the board is level (Roll and Pitch both within +/- 3 degrees of 0).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Yaw drifts rapidly even when the sensor is static | Gyro bias / offset | Apply a calibration routine or increase the deadband filter from `0.5` to `1.5` in code. |
| Serial output reads NaN | Sensor reading error | Check I2C bus wiring and verify the sensor is initialized. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [94 - MPU-6050 Gyroscope Position HUD](../intermediate/94-mpu-6050-gyroscope-position-hud.md)
- [155 - OLED Complementary Filter Balancer](155-oled-complementary-filter.md)
