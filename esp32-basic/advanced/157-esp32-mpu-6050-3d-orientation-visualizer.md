# 157 - ESP32 MPU-6050 3D Orientation Visualizer

Build a 3D attitude tracking node that fuses raw accelerometer and gyroscope data from an MPU-6050 sensor using a Complementary Filter, streaming structured serial data packets to run an external 3D visualizer.

## Goal
Learn how to implement dual-axis complementary filter calculations, pack multiple float values into serial data frames, and design coordinates streaming interfaces.

## What You Will Build
An MPU-6050 sensor is connected to I2C (GPIO 21/22). The ESP32 calculates pitch and roll angles using a Complementary Filter at 100Hz. The angles are formatted and printed to the USB Serial port as `ORIENTATION:pitch,roll,yaw` strings. This serial stream can be parsed by external scripts (Processing or Python) to rotate a 3D block model in real time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MPU-6050 6-DOF IMU Sensor | `mpu6050` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C shared lines |
| MPU-6050 | VCC / GND | 3V3 / GND | Red / Black | IMU Power |

> **Wiring tip:** Standard I2C pins connect the IMU to the ESP32. Keep the board flat at boot to calibrate gyroscope offsets.

## Code
```cpp
// MPU-6050 3D Orientation Visualizer (Serial Data Streaming)
#include <Wire.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <math.h>

Adafruit_MPU6050 mpu;

// Complementary Filter constants
const float ALPHA = 0.98;

float pitch = 0.0;
float roll = 0.0;
float yaw = 0.0; // Yaw is estimated from gyro only and will experience slow drift

unsigned long lastTime = 0;

void setup() {
  Serial.begin(115200);
  
  if (!mpu.begin()) {
    Serial.println("MPU-6050 initialization failed!");
    while(1) {}
  }
  
  mpu.setAccelerometerRange(MPU6050_RANGE_4_G);
  mpu.setGyroRange(MPU6050_RANGE_250_DEG);
  
  // Initialize filter variables
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);
  
  pitch = atan2(-a.acceleration.x, sqrt(a.acceleration.y * a.acceleration.y + a.acceleration.z * a.acceleration.z)) * 180.0 / M_PI;
  roll  = atan2(a.acceleration.y, a.acceleration.z) * 180.0 / M_PI;
  
  lastTime = millis();
  delay(1000);
}

void loop() {
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);
  
  unsigned long now = millis();
  float dt = (now - lastTime) / 1000.0;
  if (dt <= 0.0) dt = 0.01;
  lastTime = now;
  
  // 1. Calculate raw tilt angles from Accelerometer
  float accPitch = atan2(-a.acceleration.x, sqrt(a.acceleration.y * a.acceleration.y + a.acceleration.z * a.acceleration.z)) * 180.0 / M_PI;
  float accRoll  = atan2(a.acceleration.y, a.acceleration.z) * 180.0 / M_PI;
  
  // 2. Convert gyro rates from rad/s to deg/s
  float gyroRateX = g.gyro.x * 180.0 / M_PI;
  float gyroRateY = g.gyro.y * 180.0 / M_PI;
  float gyroRateZ = g.gyro.z * 180.0 / M_PI;
  
  // 3. Apply Complementary Filter
  // Pitch (Y-axis rotation) and Roll (X-axis rotation)
  pitch = ALPHA * (pitch + gyroRateY * dt) + (1.0 - ALPHA) * accPitch;
  roll  = ALPHA * (roll + gyroRateX * dt) + (1.0 - ALPHA) * accRoll;
  
  // Yaw (Z-axis rotation) integrates gyro directly (drifts over time without magnetometer)
  yaw += gyroRateZ * dt;
  if (yaw < -180.0) yaw += 360.0;
  if (yaw > 180.0)  yaw -= 360.0;
  
  // 4. Stream structured data packets to USB Serial
  // Processing/Python visualizers parse this string using commas as delimiters
  Serial.print("ORIENTATION:");
  Serial.print(pitch, 2);  Serial.print(",");
  Serial.print(roll, 2);   Serial.print(",");
  Serial.print(yaw, 2);
  Serial.println();
  
  delay(10); // 100Hz update frequency
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **MPU-6050** onto the canvas.
2. Wire both I2C lines to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Adjust the MPU-6050 acceleration X and Y sliders. Watch the formatted `ORIENTATION:[pitch],[roll],[yaw]` coordinates stream in the Serial Monitor.

## Expected Output
Serial Monitor:
```
ORIENTATION:0.00,0.00,0.00
ORIENTATION:-12.45,4.32,1.20
ORIENTATION:-25.80,15.65,2.45
```

## Expected Canvas Behavior
* At boot, the IMU initializes.
* Adjusting the simulated IMU sliders outputs matching formatted data packets on the Serial Monitor at 100Hz.
* The yaw value increments when the gyro Z-axis slider is offset.

## Code Walkthrough
| Line | Math / Action |
| --- | --- |
| `pitch = ALPHA * ...` | Calculates complementary filter Pitch values, fusing Y gyroscope rates and X accelerometer tilt. |
| `yaw += gyroRateZ * dt` | Integrates the Z gyroscope rate to estimate yaw orientation. |
| `Serial.print("ORIENTATION:")` | Appends a header tag so the receiving script can identify telemetry packets easily. |

## Hardware & Safety Concept: Serial Data Parsing and 3D Visualizer Interfaces
3D orientation visualizers (written in Processing or Python) read serial data from the USB port. The script scans for the header tag (`ORIENTATION:`), splits the remaining string using commas as delimiters, and converts the substrings into floating-point variables. These variables are then passed to 3D rotation functions (`rotateX(roll)`, `rotateY(pitch)`) to render a 3D block in sync with the physical IMU.

## Try This! (Challenges)
1. **Calibration routine offset**: Write a routine that averages 100 samples on startup to calculate and subtract gyro bias offsets automatically.
2. **Telemetry Status LED**: Flash an LED on GPIO 12 on transmissions.
3. **Data packet checksum**: Add a simple checksum (the sum of the values) to the end of the packet to verify data integrity.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Telemetry lags behind movements | Delay in loop is too large | Keep the loop delay to 10 ms (100Hz) to match the visualizer's refresh rate |
| Yaw drifts continuously | Normal gyro integration drift | A gyroscope measures rate, not absolute angle. Yaw will drift over time without a magnetometer (HMC5883L) |
| Output shows garbled characters | Baud rate mismatch | Ensure the Serial Monitor baud rate matches `115200` |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [93 - ESP32 MPU-6050 Accelerometer Serial Logs](../intermediate/93-esp32-mpu6050-accelerometer-serial-logs.md)
- [149 - ESP32 I2C OLED Tilt-compensated Compass](149-esp32-i2c-oled-tilt-compensated-compass.md)
- [154 - ESP32 OLED Kalman Filter Demo](154-esp32-oled-kalman-filter-demo.md)
