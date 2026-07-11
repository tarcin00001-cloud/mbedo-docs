# 155 - OLED Complementary Filter Balancer

Combine MPU-6050 accelerometer and gyroscope readings using a complementary filter to compute a stable, drift-free tilt angle and display it on an SSD1306 OLED screen.

## Goal
Learn how to implement a complementary filtering equation in C++ to fuse high-frequency gyroscope rate data with stable low-frequency accelerometer gravity data without loops or arrays.

## What You Will Build
An attitude estimator. By combining gyroscope readings (which are fast but drift over time) and accelerometer readings (which are stable but sensitive to vibrations), this system calculates an accurate Pitch tilt angle. The resulting angle is rendered onto an SSD1306 OLED display.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MPU-6050 Accelerometer & Gyroscope | `mpu6050` | Yes | Yes |
| 0.96" SSD1306 OLED Display | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 | VCC | 3V3 | Red | Power supply (3.3V) |
| MPU-6050 | GND | GND | Black | Ground reference |
| MPU-6050 | SDA | SDA0 (GP17) | Blue | I2C Data line |
| MPU-6050 | SCL | SCL0 (GP16) | Yellow | I2C Clock line |
| SSD1306 OLED | VCC | 3V3 | Red | Power supply (3.3V) |
| SSD1306 OLED | GND | GND | Black | Ground reference |
| SSD1306 OLED | SDA | SDA0 (GP17) | Blue | Shared I2C Data |
| SSD1306 OLED | SCL | SCL0 (GP16) | Yellow | Shared I2C Clock |

> **Wiring tip:** Ground pins of both I2C devices must be tied directly back to the ARIES board ground to maintain I2C logic level references.

## Code
```cpp
// OLED Complementary Filter Balancer - VEGA ARIES v3
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);
Adafruit_MPU6050 mpu;

bool mpuOk = true;
bool oledOk = true;

// Complementary Filter Constants
const float ALPHA = 0.96; // Filter weight split (0.96 Gyro, 0.04 Accel)
float compAngle = 0.0;    // Calculated complementary angle (degrees)
unsigned long lastTime = 0;

void setup() {
  Serial.begin(115200);
  Wire.begin();

  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED init failed!");
    oledOk = false;
  }

  if (!mpu.begin()) {
    Serial.println("MPU-6050 init failed!");
    mpuOk = false;
  } else {
    mpu.setAccelerometerRange(MPU6050_RANGE_2_G);
    mpu.setGyroRange(MPU6050_RANGE_250_DEG);
    mpu.setFilterBandwidth(MPU6050_BAND_21_HZ);
  }

  if (oledOk) {
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 0);
    display.print("Comp Filter Init");
    display.display();
  }

  lastTime = millis();
}

void loop() {
  if (!mpuOk || !oledOk) {
    delay(1000);
    return;
  }

  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);

  unsigned long currentTime = millis();
  unsigned long elapsed = currentTime - lastTime;

  // Run filter update at 20Hz (every 50ms)
  if (elapsed >= 50) {
    float dt = (float)elapsed / 1000.0;
    lastTime = currentTime;

    // 1. Calculate pitch angle from raw accelerometer data
    // atan2 gives radians, multiply by 180 / PI to convert to degrees
    float accelAngle = atan2(a.acceleration.y, a.acceleration.z) * 180.0 / PI;

    // 2. Gyro rate is in rad/s, convert to deg/s
    float gyroRate = g.gyro.x * 180.0 / PI;

    // 3. FUSE values: complementary filter equation
    compAngle = ALPHA * (compAngle + gyroRate * dt) + (1.0 - ALPHA) * accelAngle;

    // --- OLED UPDATE ---
    display.clearDisplay();
    display.setTextSize(1);
    display.setCursor(0, 0);
    display.print("COMPLEMENTARY FILTER");

    display.setCursor(0, 20);
    display.print("Accel Angle: ");
    display.print(accelAngle, 1);
    display.print(" deg");

    display.setCursor(0, 40);
    display.print("Fused Angle: ");
    display.print(compAngle, 1);
    display.print(" deg");

    display.display();

    // Send metrics to Serial for comparison
    Serial.print("Accel:");
    Serial.print(accelAngle, 2);
    Serial.print(",GyroRate:");
    Serial.print(gyroRate, 2);
    Serial.print(",Fused:");
    Serial.println(compAngle, 2);
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **MPU-6050**, and **SSD1306 OLED** onto the canvas.
2. Wire the MPU-6050: **VCC** to **3V3**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
3. Wire the OLED: **VCC** to **3V3**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
4. Copy the code and paste it into the editor.
5. Click **Run**.
6. Rotate the simulated MPU-6050. Check the Serial Plotter to see how the fused angle stays clean compared to raw accelerometer spikes.

## Expected Output
Serial Monitor:
```
Accel:15.34,GyroRate:-45.20,Fused:14.92
Accel:15.82,GyroRate:-2.50,Fused:15.11
Accel:16.12,GyroRate:1.80,Fused:15.35
```

## Expected Canvas Behavior
* The OLED shows the static title `COMPLEMENTARY FILTER`.
* If you apply a sharp shake to the MPU-6050 widget, the `Accel Angle` jumps wildly, but the `Fused Angle` remains smooth and returns to rest stably.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `atan2(a.acceleration.y, a.acceleration.z)` | Calculates tilt angle from the relative direction of gravitational force. |
| `g.gyro.x * 180.0 / PI` | Converts the raw gyroscope roll rate from radians/second to degrees/second. |
| `compAngle = ALPHA * ...` | Blends historical integrated gyro velocity with immediate absolute gravity angle. |
| `dt` | Time interval (seconds) between successive sensor updates. |
| `display.print(compAngle, 1)` | Outputs the filtered pitch orientation angle rounded to one decimal place. |

## Hardware & Safety Concept
* **Gyroscope Drift vs Accelerometer Noise**: Gyroscopes are highly accurate over fractions of a second but accumulate integration errors (drift) over time. Accelerometers do not drift but register vibration as linear acceleration, corrupting the angle estimate. The complementary filter functions as a high-pass filter on the gyro and a low-pass filter on the accelerometer.
* **Sampling Consistency**: The time step `dt` is computed from the system clock using `millis()`. Hardcoding `dt` instead of calculating it can cause filter errors if other code delays execution.

## Try This! (Challenges)
1. **Dynamic Weighting**: Add a slide potentiometer to `ADC0`. Map it so that the user can tune `ALPHA` on the fly between 0.80 and 0.99.
2. **Horizontal Level Indicator**: Draw a horizontal line using `display.drawLine()` that shifts up and down on the OLED based on the filtered tilt angle.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Fused angle drifts off indefinitely | Gyroscope axis mismatch | Ensure `g.gyro.x` matches the tilt plane of the accelerometer pitch calculation (`atan2(y, z)`). |
| The display remains stuck at 0.0 | I2C devices frozen or unconnected | Power-cycle the simulated workspace and check that both devices share VCC and GND. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [60 - OLED SSD1306 Display Setup (I2C)](../intermediate/60-oled-ssd1306-display-setup.md)
- [154 - OLED Kalman Filter Demo](154-oled-kalman-filter.md)
