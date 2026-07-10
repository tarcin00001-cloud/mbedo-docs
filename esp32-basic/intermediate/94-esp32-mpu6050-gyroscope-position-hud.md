# 94 - ESP32 MPU-6050 Gyroscope Position HUD

Read angular velocity from an MPU-6050 gyroscope sensor and display live Pitch and Roll angles on a 16x2 I2C LCD screen.

## Goal
Learn how to calculate pitch and roll orientation angles using gravity vector components from the accelerometer, and update an LCD HUD.

## What You Will Build
An MPU-6050 sensor reads orientation data, and a 16x2 I2C LCD displays the calculated Pitch angle in degrees on Row 1 and the Roll angle on Row 2, updated every 200 ms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MPU-6050 Accelerometer & Gyroscope | `mpu6050` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 Module | VCC | 3V3 | Red | Power |
| MPU-6050 Module | GND | GND | Black | Ground |
| MPU-6050 Module | SDA | GPIO21 | Blue | I2C Data (shared) |
| MPU-6050 Module | SCL | GPIO22 | Yellow | I2C Clock (shared) |
| I2C LCD | VCC | 5V (Vin) | Red | LCD Power |
| I2C LCD | GND | GND | Black | Ground |
| I2C LCD | SDA | GPIO21 | Blue | I2C Data (shared) |
| I2C LCD | SCL | GPIO22 | Yellow | I2C Clock (shared) |

> **Wiring tip:** Share the I2C bus pins in parallel (GPIO 21 and 22) for both the MPU-6050 and the I2C LCD.

## Code
```cpp
// MPU-6050 Gyroscope Position HUD
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <math.h>

Adafruit_MPU6050 mpu;
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(115200);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("MPU-6050 Init...");
  
  if (!mpu.begin()) {
    lcd.setCursor(0, 1);
    lcd.print("Sensor error!  ");
    while (1) {}
  }
  
  delay(1000);
  lcd.clear();
}

void loop() {
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);
  
  // Calculate Pitch and Roll angles in degrees from accelerometer data
  // Pitch = atan2(-Ax, sqrt(Ay^2 + Az^2)) * 180 / PI
  // Roll = atan2(Ay, Az) * 180 / PI
  float Ax = a.acceleration.x;
  float Ay = a.acceleration.y;
  float Az = a.acceleration.z;
  
  float pitch = atan2(-Ax, sqrt(Ay * Ay + Az * Az)) * 57.2958; // 180 / PI = 57.2958
  float roll = atan2(Ay, Az) * 57.2958;
  
  // Display Pitch on Row 1
  lcd.setCursor(0, 0);
  lcd.print("Pitch: ");
  lcd.print(pitch, 1);
  lcd.print((char)223); // Degree symbol
  lcd.print("     ");
  
  // Display Roll on Row 2
  lcd.setCursor(0, 1);
  lcd.print("Roll:  ");
  lcd.print(roll, 1);
  lcd.print((char)223);
  lcd.print("     ");
  
  Serial.print("Pitch: "); Serial.print(pitch, 1);
  Serial.print(" | Roll: "); Serial.println(roll, 1);
  
  delay(200); // 5Hz update rate
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MPU-6050**, and **16x2 I2C LCD** onto the canvas.
2. Connect SDA to **GPIO21** and SCL to **GPIO22** for both components.
3. Paste the code and click **Run**.
4. Adjust the rotational pitch/roll sliders on the MPU-6050 component widget and observe the angles update on the LCD.

## Expected Output
Serial Monitor:
```
Pitch: 0.0 | Roll: 0.0
Pitch: 15.4 | Roll: -2.3
Pitch: -45.1 | Roll: 12.0
```

LCD Display (resting at minor tilt):
```
Pitch: 15.4°
Roll:  -2.3°
```

## Expected Canvas Behavior
* The LCD displays the Pitch angle on line 1 and Roll angle on line 2.
* Rotating or tilting the virtual sensor widget instantly changes the displayed angle values on the LCD.
* Calculations run continuously.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `atan2(...)` | Trigonometric function used to calculate angles over a full 360° circle. |
| `57.2958` | Mathematical constant conversion factor to translate Radians to Degrees. |
| `a.acceleration` | Retrieves the three-axis gravity vectors used for calculation. |

## Hardware & Safety Concept: Attitude and Heading Reference Systems (AHRS)
Calculating pitch and roll using acceleration gravity vectors works well when the sensor is stationary (static orientation). In dynamic environments (e.g. flight controllers on drones or balance boards), physical movement forces create noise on the acceleration data. In those systems, developers combine the accelerometer with gyroscope angular velocity readings using a **Complementary Filter** or **Kalman Filter** to get clean, stable orientation estimates.

## Try This! (Challenges)
1. **Raw Gyro velocity Display**: Add a button on GPIO 4. When active, display raw gyroscope spin velocity (deg/s) on the LCD instead of angles.
2. **Horizon Indicator**: Create a simple graphical marker on the screen that moves left/right representing roll balance.
3. **Calibrate Offset button**: Add a button on GPIO 15 that offsets the current orientation to 0 (useful for leveling).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Angle numbers jitter rapidly | Sensor noise | Implement a simple software low-pass filter on the pitch/roll variables |
| Readings show NaN | Divide by zero error if Ay and Az are both exactly 0 | Standard atan2 handles this, verify sensor is connected and transmitting data |
| Pitch/Roll values are inverted | Axis orientations swapped | Modify the sign (plus/minus) of Ax, Ay, or Az in the formulas |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [93 - ESP32 MPU-6050 Accelerometer Serial Logs](93-esp32-mpu6050-accelerometer-serial-logs.md)
- [95 - ESP32 MPU-6050 Tilt Sensor Alarm](95-esp32-mpu6050-tilt-sensor-alarm.md)
- [58 - ESP32 16×2 I2C LCD Print Text](58-esp32-16x2-i2c-lcd-print-text.md)
