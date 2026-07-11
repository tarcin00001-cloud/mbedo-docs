# 149 - ESP32 I2C OLED Tilt-compensated Compass

Build a high-precision navigation system that combines an MPU-6050 IMU and an HMC5883L magnetometer, using pitch and roll vectors to correct magnetic readings when tilted and displaying headings on an OLED.

## Goal
Learn how to apply Euler angle rotation transformations (pitch and roll) to tilt-compensate magnetic vectors, correcting digital compass readings during physical tilt movements.

## What You Will Build
An MPU-6050 sensor, HMC5883L magnetometer, and SSD1306 OLED screen share the I2C bus (GPIO 21/22). The ESP32 measures pitch and roll from the IMU, uses these angles to project the magnetic field vector onto the horizontal plane, and calculates the corrected heading, rendering it on the OLED display.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HMC5883L 3-Axis Magnetometer | `magnetometer` | Yes | Yes |
| MPU-6050 6-DOF IMU Sensor | `mpu6050` | Yes | Yes |
| 0.96″ SSD1306 OLED (I2C) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C shared lines |
| MPU-6050 | VCC / GND | 3V3 / GND | Red / Black | IMU Power |
| HMC5883L | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C shared lines |
| HMC5883L | VCC / GND | 3V3 / GND | Red / Black | Magnetometer Power |
| SSD1306 OLED | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C shared lines |
| SSD1306 OLED | VCC / GND | 3V3 / GND | Red / Black | OLED Power |

> **Wiring tip:** Share the SDA (GPIO 21) and SCL (GPIO 22) lines. Keep I2C wire lengths short (under 20 cm) or add 4.7 kΩ pull-up resistors to maintain signal integrity with multiple devices on the bus.

## Code
```cpp
// I2C OLED Tilt-compensated Compass (MPU-6050 + HMC5883L + OLED)
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <math.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C
#define HMC5883L_ADDR 0x1E

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);
Adafruit_MPU6050 mpu;

// Declination angle at your location (Bangalore = +0.023 radians)
const float DECLINATION_ANGLE = 0.023;

void initHMC5883L() {
  Wire.beginTransmission(HMC5883L_ADDR);
  Wire.write(0x02); // Mode register
  Wire.write(0x00); // Continuous measurement
  Wire.endTransmission();
}

void readMagnetometer(int &x, int &y, int &z) {
  Wire.beginTransmission(HMC5883L_ADDR);
  Wire.write(0x03); // Data registers MSB
  Wire.endTransmission();
  
  Wire.requestFrom(HMC5883L_ADDR, 6);
  if (Wire.available() >= 6) {
    x = (Wire.read() << 8) | Wire.read();
    y = (Wire.read() << 8) | Wire.read();
    z = (Wire.read() << 8) | Wire.read();
  }
}

void setup() {
  Serial.begin(115200);
  
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED init failed!");
    while(1) {}
  }
  
  if (!mpu.begin()) {
    Serial.println("MPU-6050 init failed!");
    while(1) {}
  }
  
  initHMC5883L();
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 20);
  display.print("Calibrating IMU...");
  display.display();
  delay(1500);
}

void loop() {
  // 1. Read IMU Pitch and Roll
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);
  
  float ax = a.acceleration.x;
  float ay = a.acceleration.y;
  float az = a.acceleration.z;
  
  // Calculate Roll (phi) and Pitch (theta)
  float roll  = atan2(ay, az);
  float pitch = atan2(-ax, sqrt(ay * ay + az * az));
  
  // 2. Read raw Magnetometer values
  int mx, my, mz;
  readMagnetometer(mx, my, mz);
  
  // 3. Apply Tilt-Compensation Formulas
  // Project magnetic vectors (mx, my, mz) onto the horizontal plane
  float cosRoll  = cos(roll);
  float sinRoll  = sin(roll);
  float cosPitch = cos(pitch);
  float sinPitch = sin(pitch);
  
  float mxComp = mx * cosPitch + mz * sinPitch;
  float myComp = mx * sinRoll * sinPitch + my * cosRoll - mz * sinRoll * cosPitch;
  
  // 4. Calculate Corrected Heading
  float heading = atan2(myComp, mxComp);
  heading += DECLINATION_ANGLE;
  
  // Normalize to 0-360 degrees
  if (heading < 0) heading += 2 * M_PI;
  if (heading > 2 * M_PI) heading -= 2 * M_PI;
  
  float headingDegrees = heading * 180.0 / M_PI;
  
  // Display Results
  display.clearDisplay();
  
  display.setCursor(0, 0);
  display.setTextSize(1);
  display.print("TILT-COMP COMPASS");
  display.drawFastHLine(0, 10, 128, SSD1306_WHITE);
  
  display.setCursor(0, 15);
  display.print("Pitch: "); display.print(pitch * 180.0 / M_PI, 0); display.print(" deg");
  display.setCursor(0, 25);
  display.print("Roll:  "); display.print(roll * 180.0 / M_PI, 0); display.print(" deg");
  
  display.setTextSize(2);
  display.setCursor(0, 45);
  display.print((int)headingDegrees);
  display.print((char)247);
  display.print(" ");
  
  // Display compass arrow needle (centered at 96, 42 radius 15)
  int cx = 96, cy = 42, r = 15;
  display.drawCircle(cx, cy, r, SSD1306_WHITE);
  int nx = cx + (int)(r * sin(-heading));
  int ny = cy - (int)(r * cos(-heading));
  display.drawLine(cx, cy, nx, ny, SSD1306_WHITE);
  
  display.display();
  
  Serial.print("Heading: "); Serial.println(headingDegrees, 1);
  
  delay(100);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MPU-6050**, **HMC5883L**, and **SSD1306 OLED** onto the canvas.
2. Wire shared I2C SDA/SCL lines to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the MPU-6050 X and Y acceleration vectors (tilting the device). Notice the pitch and roll update on the OLED screen, while the heading needle corrects its vector angle.

## Expected Output
Serial Monitor:
```
Heading: 180.5
Heading: 180.2 (Stays constant even when pitch/roll sliders change!)
```

OLED Screen:
```
TILT-COMP COMPASS
─────────────────
Pitch: 15 deg
Roll:  -5 deg
180°       ┌───┐
           │ / │
           └───┘
```

## Expected Canvas Behavior
* The LCD prints active Pitch/Roll and corrected headings.
* Tilting the IMU widget slider updates the pitch/roll readings.
* The heading needle remains accurate even when the IMU is tilted.

## Code Walkthrough
| Line | Check / Math |
| --- | --- |
| `roll = atan2(ay, az)` | Computes the roll angle around the longitudinal axis. |
| `mxComp = mx * cosPitch + ...` | Projects the X magnetometer component onto the horizontal plane. |
| `myComp = mx * sinRoll * ...` | Projects the Y magnetometer component, correcting for both roll and pitch. |

## Hardware & Safety Concept: Gimbal Lock and Tilt Compensation
In a simple digital compass, tilting the sensor changes the direction of the X and Y axes relative to the Earth's surface, corrupting the heading calculation. To correct this, an IMU (Inertial Measurement Unit) measures the gravity vector to calculate pitch and roll. A **rotation matrix** is then applied to the magnetometer readings, projecting them back onto a virtual flat horizontal plane before calculating the heading. This ensures heading calculations remain accurate during turns and climbs (e.g. in quadcopters or marine vessels).

## Try This! (Challenges)
1. **Dynamic Heading Lock**: Add a button on GPIO 4. When pressed, store the current heading as a target course, and sound a buzzer if the robot drifts off course by more than 15 degrees.
2. **Complementary Filter Integration**: Smooth out the accelerometer readings using a complementary filter with gyroscope data to make tilt calculations less sensitive to vibration.
3. **Roll Warning Flasher**: Display a warning screen on the OLED if the roll angle exceeds 45 degrees.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Heading drifts wildly during tilts | Axis orientations mismatched | Verify that both sensors share the same coordinate frame (check sensor datasheet markings) |
| Compass needle gets stuck | I2C bus congestion | Ensure pull-up resistors are installed and address pins are correct |
| Readouts are slow | heavy float math | Set the serial baud rate to 115200 and optimize I2C speed to 400 kHz |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [94 - ESP32 MPU-6050 Gyroscope Position HUD](../intermediate/94-esp32-mpu6050-gyroscope-position-hud.md)
- [148 - ESP32 I2C OLED Digital Compass](148-esp32-i2c-oled-digital-compass.md)
- [154 - ESP32 OLED Kalman Filter Demo](154-esp32-oled-kalman-filter-demo.md)
