# 94 - Pico MPU6050 HUD

Display raw MPU-6050 motion tracking telemetry on a 16x2 character LCD screen.

## Goal
Learn how to read high-speed I2C sensors and display multi-axis coordinates formatted on character screens.

## What You Will Build
An inertial navigation display panel:
- **MPU-6050 IMU (GP4 SDA, GP5 SCL)**: Monitors tilt rates.
- **16x2 I2C LCD**: Displays X and Y axis acceleration values (e.g. "X: 0 | Y: 120") on row 0, and Z-axis values on row 1.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MPU-6050 IMU Module | `mpu6050` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MPU-6050 | VCC | 3.3V | Power supply |
| MPU-6050 | SDA | GP4 | Shared I2C Data |
| MPU-6050 | SCL | GP5 | Shared I2C Clock |
| MPU-6050 | GND | GND | Ground return |
| I2C LCD | VCC | 5V | LCD Power |
| I2C LCD | SDA | GP4 | Shared I2C Data |
| I2C LCD | SCL | GP5 | Shared I2C Clock |
| I2C LCD | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// Raw shims required by interpreted mode
int16_t Ac_x, Ac_y, Ac_z;
int16_t Gy_x, Gy_y, Gy_z;

const int MPU_addr = 0x68;
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  // Wake up MPU-6050
  Wire.beginTransmission(MPU_addr);
  Wire.write(0x6B);
  Wire.write(0);
  Wire.endTransmission(true);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("IMU Status HUD");
  delay(1000);
}

void loop() {
  Wire.beginTransmission(MPU_addr);
  Wire.write(0x3B);
  Wire.endTransmission(false);
  Wire.requestFrom(MPU_addr, 14, true);

  // Read raw acceleration
  Ac_x = Wire.read() << 8 | Wire.read();
  Ac_y = Wire.read() << 8 | Wire.read();
  Ac_z = Wire.read() << 8 | Wire.read();
  
  // Skip temp bytes
  int16_t tmp = Wire.read() << 8 | Wire.read();

  // Read raw gyro
  Gy_x = Wire.read() << 8 | Wire.read();
  Gy_y = Wire.read() << 8 | Wire.read();
  Gy_z = Wire.read() << 8 | Wire.read();

  lcd.clear();

  // Row 0: X and Y axis acceleration values
  lcd.setCursor(0, 0);
  lcd.print("X:");
  lcd.print(Ac_x / 100);
  lcd.print("  Y:");
  lcd.print(Ac_y / 100);

  // Row 1: Z axis acceleration
  lcd.setCursor(0, 1);
  lcd.print("Z:");
  lcd.print(Ac_z / 100);
  lcd.print(" (x100 LSB)");

  delay(300); // 3 refreshes per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MPU-6050 Sensor**, and **I2C LCD** onto the canvas.
2. Connect both MPU-6050 and LCD to GP4 (SDA) and GP5 (SCL) in parallel.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the rotation of the MPU-6050 on canvas and watch the values display on the LCD.

## Expected Output

Terminal:
```
Simulation active. LCD rendering IMU coordinates.
```

## Expected Canvas Behavior
* Row 0: `X:0  Y:0` (when flat)
* Row 1: `Z:163 (x100 LSB)`

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Ac_x / 100` | Scales raw large acceleration integers down to fit the limited screen space of the 16x2 character LCD columns. |

## Hardware & Safety Concept: Display Space Optimization
Standard character LCDs are limited to 16 columns of text. When displaying high-resolution sensor metrics (such as 16-bit integers from the MPU-6050 which range from -32768 to 32768), formatting space becomes tight. Scaling down the numbers (e.g. dividing by 100) or printing compact labels (e.g. "X:" instead of "X-Axis:") is necessary to fit all coordinate readings on a single row.

## Try This! (Challenges)
1. **Dynamic Tilt Alarm**: Flash the LCD backlight ON and OFF if the X-axis value `Ac_x` exceeds 9000 (indicating >45 degrees tilt).
2. **Gyro Display**: Modify the code to display gyroscope rotation rates `Gy_x` and `Gy_y` on the screen instead of acceleration.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD numbers display overlapping remnants | Screen clear missing | Verify `lcd.clear()` is executed at the start of each display loop cycle. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode. Variables representing MPU-6050 inputs must be named exactly `Ac_x`, `Ac_y`, `Ac_z`, `Gy_x`, `Gy_y`, `Gy_z`.

## Related Projects
- [58 - Pico LCD Print](58-pico-lcd-print.md)
- [93 - Pico MPU6050 Serial](93-pico-mpu6050-serial.md)
