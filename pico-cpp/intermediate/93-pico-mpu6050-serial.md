# 93 - Pico MPU6050 Serial

Read and log raw accelerometer and gyroscope data from an MPU-6050 sensor to the Serial Monitor.

## Goal
Learn how to interface 6-axis motion tracking sensors (IMUs) over I2C and read raw acceleration and angular velocity values.

## What You Will Build
An IMU logger:
- **MPU-6050 IMU (GP4 SDA, GP5 SCL)**: Measures raw acceleration ($Ac_x, Ac_y, Ac_z$) and gyroscope ($Gy_x, Gy_y, Gy_z$) values, logging them to the Serial Monitor every 500 ms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MPU-6050 IMU Module | `mpu6050` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MPU-6050 | VCC | 3.3V | Power supply |
| MPU-6050 | SDA | GP4 (I2C0 SDA) | I2C Data line |
| MPU-6050 | SCL | GP5 (I2C0 SCL) | I2C Clock line |
| MPU-6050 | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>

// MPU-6050 raw variable shims required by MbedO interpreter
int16_t Ac_x, Ac_y, Ac_z;
int16_t Gy_x, Gy_y, Gy_z;

const int MPU_addr = 0x68;

void setup() {
  Serial.begin(9600);
  
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();
  
  // Wake up MPU-6050 (write 0 to power management register 6B)
  Wire.beginTransmission(MPU_addr);
  Wire.write(0x6B);
  Wire.write(0);
  Wire.endTransmission(true);

  Serial.println("MPU-6050 Logger Online");
}

void loop() {
  Wire.beginTransmission(MPU_addr);
  Wire.write(0x3B); // Start reading from register 3B (ACCEL_XOUT_H)
  Wire.endTransmission(false);
  Wire.requestFrom(MPU_addr, 14, true); // Request 14 registers

  // Read raw acceleration data (high and low bytes combined)
  Ac_x = Wire.read() << 8 | Wire.read();
  Ac_y = Wire.read() << 8 | Wire.read();
  Ac_z = Wire.read() << 8 | Wire.read();
  
  // Skip temperature registers (high and low bytes combined)
  int16_t tmp = Wire.read() << 8 | Wire.read();

  // Read raw gyroscope data (high and low bytes combined)
  Gy_x = Wire.read() << 8 | Wire.read();
  Gy_y = Wire.read() << 8 | Wire.read();
  Gy_z = Wire.read() << 8 | Wire.read();

  // Print raw values to Serial Monitor
  Serial.print("Accel: ");
  Serial.print(Ac_x); Serial.print(", ");
  Serial.print(Ac_y); Serial.print(", ");
  Serial.print(Ac_z);
  Serial.print(" | Gyro: ");
  Serial.print(Gy_x); Serial.print(", ");
  Serial.print(Gy_y); Serial.print(", ");
  Serial.println(Gy_z);

  delay(500);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **MPU-6050 Sensor** onto the canvas.
2. Connect MPU-6050: **VCC** to **3.3V**, **SDA** to **GP4**, **SCL** to **GP5**, **GND** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the accelerometer/gyro rotation settings on canvas and monitor the changing values in the Serial console.

## Expected Output

Terminal:
```
MPU-6050 Logger Online
Accel: 0, 0, 16384 | Gyro: 0, 0, 0
Accel: 4096, -2048, 15800 | Gyro: 150, -50, 20
```

## Expected Canvas Behavior
| MPU-6050 Rotation | `Ac_x` (Accel X) | `Ac_y` (Accel Y) | `Ac_z` (Accel Z) |
| --- | --- | --- | --- |
| Flat (Normal) | ~0 | ~0 | ~16384 (+1g) |
| Tilted Left | > 4000 | ~0 | < 16384 |
| Tilted Forward | ~0 | > 4000 | < 16384 |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Wire.write(0x3B)` | Tells the MPU-6050 internal register index to target Accel X High register (offset 0x3B) for read. |
| `Wire.read() << 8 \| Wire.read()` | Reads two consecutive bytes from the register and bit-shifts/OR-merges them into a single 16-bit signed integer. |

## Hardware & Safety Concept: Sensor Scale Ranges
The MPU-6050 is a micro-electromechanical system (MEMS) chip containing tiny capacitive structures that move under physical acceleration. Its default scale range is set to $\pm 2g$ for the accelerometer, yielding a sensitivity of 16384 LSB/g (hence why gravity reads as `16384` on the Z-axis when flat). The gyroscope defaults to $\pm 250$ degrees per second (°/s). You can change these ranges in registers to adjust sensitivity.

## Try This! (Challenges)
1. **Activity Warning**: Add an LED on GP15 and flash it rapidly only if the gyroscope readings `Gy_x` or `Gy_y` exceed 5000 (indicating fast rotation).
2. **Tilt Alert**: Light a red LED on GP14 if the X-axis acceleration `Ac_x` exceeds 8000.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| All readings show `0` or `-1` | I2C address error | Verify that the MPU-6050 is powered and that SDA/SCL lines are not swapped. Standard address is 0x68, but pulling the AD0 pin high shifts it to 0x69. |

## Mode Notes
This basic I2C sensor project runs locally and is supported by MbedO interpreted mode. Variables representing MPU-6050 accelerometer inputs must be named exactly `Ac_x`, `Ac_y`, `Ac_z`, `Gy_x`, `Gy_y`, `Gy_z` to be resolved by the shims.

## Related Projects
- [94 - Pico MPU6050 HUD](94-pico-mpu6050-hud.md)
- [95 - Pico MPU6050 Tilt](95-pico-mpu6050-tilt.md)
