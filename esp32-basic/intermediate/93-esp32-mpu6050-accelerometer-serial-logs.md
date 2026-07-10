# 93 - ESP32 MPU-6050 Accelerometer Serial Logs

Read raw three-axis acceleration data (X, Y, Z axes) from an MPU-6050 inertial sensor and print the log metrics to the Serial Monitor.

## Goal
Learn how to interface I2C inertial sensors, initialise the MPU-6050 chip using the Adafruit MPU6050 library, and read acceleration forces in meters per second squared (\(m/s^2\)).

## What You Will Build
An MPU-6050 sensor connected to the ESP32's I2C bus (SDA: GPIO 21, SCL: GPIO 22). The ESP32 queries the sensor's accelerometer every 500 ms and outputs the raw X, Y, and Z acceleration forces to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MPU-6050 Accelerometer & Gyroscope | `mpu6050` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 Module | VCC | 3V3 | Red | Power (3.3V compatible) |
| MPU-6050 Module | GND | GND | Black | Ground reference |
| MPU-6050 Module | SDA | GPIO21 | Blue | I2C Data line |
| MPU-6050 Module | SCL | GPIO22 | Yellow | I2C Clock line |

> **Wiring tip:** The MPU-6050 is an I2C sensor. Wire SDA to GPIO 21 and SCL to GPIO 22. Standard breakout modules contain pull-up resistors on the I2C lines.

## Code
```cpp
// MPU-6050 Accelerometer Serial Logs
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <Wire.h>

Adafruit_MPU6050 mpu;

void setup() {
  Serial.begin(115200);
  Serial.println("Initialising MPU-6050 sensor...");
  
  if (!mpu.begin()) {
    Serial.println("Failed to find MPU-6050 chip! Check wiring.");
    while (1) {
      delay(10);
    }
  }
  
  Serial.println("MPU-6050 Found.");
  
  // Set accelerometer range (+/- 2g, 4g, 8g, or 16g)
  mpu.setAccelerometerRange(MPU6050_RANGE_8_G);
  
  // Set filter bandwidth
  mpu.setFilterBandwidth(MPU6050_BAND_21_HZ);
}

void loop() {
  // Read sensor events
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);
  
  // Print Acceleration values in m/s^2
  Serial.print("Accel X: ");
  Serial.print(a.acceleration.x, 2);
  Serial.print(" | Y: ");
  Serial.print(a.acceleration.y, 2);
  Serial.print(" | Z: ");
  Serial.print(a.acceleration.z, 2);
  Serial.println(" m/s^2");
  
  delay(500); // 2Hz refresh rate
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **MPU-6050** onto the canvas.
2. Wire SDA to **GPIO21** and SCL to **GPIO22**.
3. Paste the code and click **Run**.
4. Adjust the rotational/tilt angles of the MPU-6050 component widget on the canvas, and watch the accelerometer values change in the Serial Monitor.

## Expected Output
Serial Monitor (when resting flat on a table):
```
Initialising MPU-6050 sensor...
MPU-6050 Found.
Accel X: 0.05 | Y: -0.12 | Z: 9.81 m/s^2
Accel X: 4.82 | Y: 2.10 | Z: 8.10 m/s^2
```

## Expected Canvas Behavior
* At startup, the MPU-6050 is initialised and verified.
* If the sensor is flat (no movement), Accel Z registers gravity (\(\approx 9.81\ m/s^2\)), while X and Y stay close to 0.
* Tilting the sensor widget transfers the gravity vector component to the X and Y axes, changing the printed metrics dynamically.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `Adafruit_MPU6050 mpu` | Creates the sensor control object. |
| `mpu.setAccelerometerRange(...)` | Configures the measurement scale (e.g. 8G allows reading higher acceleration forces). |
| `mpu.getEvent(&a, &g, &temp)` | Reads raw register bytes over I2C and converts them to standard metric units. |
| `a.acceleration.x` | Accesses the calculated X-axis acceleration value. |

## Hardware & Safety Concept: Micro-Electro-Mechanical Systems (MEMS)
The MPU-6050 contains microscopic silicon structures suspended inside the chip. When acceleration forces occur (either by physical movement or gravity tilt), these structures bend slightly, changing the electrical capacitance between plates. The chip measures this change and outputs acceleration. Accel Z reads \(\approx 9.8\ m/s^2\) at rest because gravity pulls down on the internal structures.

## Try This! (Challenges)
1. **Total Acceleration magnitude**: Calculate and log the total acceleration vector magnitude: \(a_{total} = \sqrt{x^2 + y^2 + z^2}\).
2. **Fall Detector**: Print a warning "FALL DETECTED" if the total acceleration magnitude drops below 2.0 \(m/s^2\) (simulating freefall).
3. **Double Tap detection**: Log a tap event if acceleration on the X or Y axis spikes rapidly above a threshold.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| "Failed to find MPU-6050..." | SDA/SCL lines reversed or power issue | Verify SDA connects to GPIO 21, SCL to GPIO 22, VCC to 3.3V |
| Readings are static and do not change | Event reading failure | Ensure `mpu.getEvent` is called inside the loop |
| Readings drift | Natural sensor offset noise | Calibrate the sensor by subtracting the resting offset values from your raw readings |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [94 - ESP32 MPU-6050 Gyroscope Position HUD](94-esp32-mpu6050-gyroscope-position-hud.md)
- [95 - ESP32 MPU-6050 Tilt Sensor Alarm](95-esp32-mpu6050-tilt-sensor-alarm.md)
- [78 - ESP32 BMP180 Pressure Sensor Serial Logs](78-esp32-bmp180-pressure-sensor-serial-logs.md)
