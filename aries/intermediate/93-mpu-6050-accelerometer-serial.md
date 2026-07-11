# 93 - MPU-6050 Accelerometer Serial Logs

Read raw three-axis acceleration data (X, Y, Z axes) from an MPU-6050 inertial sensor and print the log metrics to the Serial Monitor.

## Goal
Learn how to interface I2C inertial sensors, initialize the MPU-6050 chip using the Adafruit MPU6050 library, and read acceleration forces in meters per second squared ($m/s^2$) without using loops inside C++ code.

## What You Will Build
An MPU-6050 sensor is connected to the ARIES v3 board's default I2C0 interface. The board queries the sensor's accelerometer every 500 ms and outputs the raw X, Y, and Z acceleration forces to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MPU-6050 Accelerometer & Gyroscope | `mpu6050` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 Module | VCC | 3V3 | Red | Power (3.3V compatible) |
| MPU-6050 Module | GND | GND | Black | Ground reference |
| MPU-6050 Module | SDA | SDA0 (GP17) | Blue | I2C Data line |
| MPU-6050 Module | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** The MPU-6050 is an I2C sensor. Wire SDA to SDA0 (GP17) and SCL to SCL0 (GP16). Standard breakout modules contain pull-up resistors on the I2C lines.

## Code
```cpp
// MPU-6050 Accelerometer Serial Logs
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <Wire.h>

Adafruit_MPU6050 mpu;
bool sensorOk = true;

void setup() {
  Serial.begin(115200);
  Wire.begin(); // Initialize default I2C0 interface
  Serial.println("Initializing MPU-6050 sensor...");
  
  if (!mpu.begin()) {
    Serial.println("Failed to find MPU-6050 chip! Check wiring.");
    sensorOk = false;
  }
  
  if (sensorOk) {
    Serial.println("MPU-6050 Found.");
    
    // Set accelerometer range (+/- 2g, 4g, 8g, or 16g)
    mpu.setAccelerometerRange(MPU6050_RANGE_8_G);
    
    // Set filter bandwidth
    mpu.setFilterBandwidth(MPU6050_BAND_21_HZ);
  }
}

void loop() {
  if (!sensorOk) {
    delay(2000);
    return; // Halt logic execution
  }
  
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
1. Drag the **VEGA ARIES v3** and **MPU-6050** components onto the canvas.
2. Wire SDA to **SDA0 (GP17)** and SCL to **SCL0 (GP16)**. Connect VCC to **3V3**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.
6. Adjust the rotational/tilt angles of the MPU-6050 component widget on the canvas, and watch the accelerometer values change in the Serial Monitor.

## Expected Output
Serial Monitor (when resting flat on a table):
```
Initializing MPU-6050 sensor...
MPU-6050 Found.
Accel X: 0.05 | Y: -0.12 | Z: 9.81 m/s^2
Accel X: 4.82 | Y: 2.10 | Z: 8.10 m/s^2
```

## Expected Canvas Behavior
* At startup, the MPU-6050 is initialized and verified.
* If the sensor is flat (no movement), Accel Z registers gravity ($\approx 9.81\ m/s^2$), while X and Y stay close to 0.
* Tilting the sensor widget transfers the gravity vector component to the X and Y axes, changing the printed metrics dynamically.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Adafruit_MPU6050 mpu` | Creates the sensor control driver object. |
| `mpu.begin()` | Starts I2C transactions and reads the device WHO_AM_I register. |
| `mpu.setAccelerometerRange(...)` | Configures the measurement scale (e.g. 8G allows reading higher acceleration forces). |
| `mpu.getEvent(&a, &g, &temp)` | Reads raw register bytes over I2C and converts them to metric units. |
| `a.acceleration.x` | Accesses the calculated X-axis acceleration value. |

## Hardware & Safety Concept
* **Micro-Electro-Mechanical Systems (MEMS)**: The MPU-6050 contains microscopic silicon structures suspended inside the chip. When acceleration forces occur (either by physical movement or gravity tilt), these structures bend slightly, changing the electrical capacitance between plates. The chip measures this change and outputs acceleration. Accel Z reads $\approx 9.8\ m/s^2$ at rest because gravity pulls down on the internal structures.

## Try This! (Challenges)
1. **Total Acceleration magnitude**: Calculate and log the total acceleration vector magnitude: $a_{total} = \sqrt{x^2 + y^2 + z^2}$ using standard `sqrt()` and math functions.
2. **Fall Detector**: Print a warning "FALL DETECTED" if the total acceleration magnitude drops below 2.0 $m/s^2$ (simulating freefall) without using loops.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| "Failed to find MPU-6050..." | SDA/SCL lines reversed | Verify SDA connects to GP17, SCL connects to GP16. |
| Readings are static and do not change | Event reading failure | Ensure `mpu.getEvent` is called inside the loop. |
| Readings drift over time | Temperature changes or noise | Subtract the resting offset values from your raw readings (calibration offset). |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [94 - MPU-6050 Gyroscope Position HUD](94-mpu-6050-gyroscope-position-hud.md)
- [95 - MPU-6050 Tilt Sensor Alarm](95-mpu-6050-tilt-sensor-alarm.md)
- [78 - BMP180 Pressure Sensor Serial Logs](78-bmp180-pressure-sensor-serial.md)
