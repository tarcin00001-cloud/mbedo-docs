# 154 - OLED Kalman Filter Demo

Smooth out noisy raw accelerometer readings from an MPU-6050 sensor using a 1D Kalman filter and display both the raw and filtered values on an SSD1306 OLED screen.

## Goal
Learn how to implement a 1D Kalman filter algorithm on the VEGA ARIES v3 board without C++ loops or array buffers, processing real-time sensor streams and updating an I2C OLED screen.

## What You Will Build
A real-time signal smoothing system. The MPU-6050 provides accelerometer data. Due to motor vibrations or hand shaking, this data is noisy. The code runs a continuous 1D Kalman filter on the raw Accel X axis, displaying the instantaneous noisy value and the smooth Kalman-filtered estimate side-by-side on the OLED display.

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

> **Wiring tip:** Since both components are I2C devices, connect their SDA and SCL pins in parallel to ARIES pins GP17 and GP16 respectively.

## Code
```cpp
// OLED Kalman Filter Demo - VEGA ARIES v3
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

// 1D Kalman Filter States
float Q = 0.022;     // Process noise covariance (estimation uncertainty)
float R = 0.617;     // Measurement noise covariance (sensor noise)
float x_est = 0.0;   // Estimated value
float P = 1.0;       // Estimation error covariance
float K = 0.0;       // Kalman gain

unsigned long lastTime = 0;

void setup() {
  Serial.begin(115200);
  Wire.begin();

  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED allocation failed!");
    oledOk = false;
  }
  
  if (!mpu.begin()) {
    Serial.println("MPU-6050 allocation failed!");
    mpuOk = false;
  } else {
    mpu.setAccelerometerRange(MPU6050_RANGE_2_G);
    mpu.setFilterBandwidth(MPU6050_BAND_5_HZ);
  }

  if (oledOk) {
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 0);
    display.print("Kalman Filter Init");
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

  float measuredVal = a.acceleration.x;

  unsigned long currentTime = millis();
  unsigned long elapsed = currentTime - lastTime;

  // Filter and display updates at 10Hz (every 100ms)
  if (elapsed >= 100) {
    lastTime = currentTime;

    // --- KALMAN FILTER ALGORITHM ---
    // 1. Prediction Update: State estimation remains unchanged (1D static system)
    // Covariance update:
    P = P + Q;

    // 2. Measurement Update: Calculate Kalman Gain
    K = P / (P + R);

    // 3. State update: Correct the estimate with the new measurement
    x_est = x_est + K * (measuredVal - x_est);

    // 4. Covariance update: Correct estimation error
    P = (1.0 - K) * P;

    // --- OLED DISPLAY RENDER ---
    display.clearDisplay();
    display.setTextSize(1);
    display.setCursor(0, 0);
    display.print("KALMAN SMOOTHING");

    display.setCursor(0, 20);
    display.print("Raw X: ");
    display.print(measuredVal, 3);
    display.print(" m/s2");

    display.setCursor(0, 40);
    display.print("Filt X: ");
    display.print(x_est, 3);
    display.print(" m/s2");

    display.display();

    // Export raw vs filtered telemetry for Serial Plotting
    Serial.print("Raw:");
    Serial.print(measuredVal, 3);
    Serial.print(",");
    Serial.print("Filtered:");
    Serial.println(x_est, 3);
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, **MPU-6050**, and **SSD1306 OLED** onto the canvas.
2. Wire the MPU-6050: **VCC** to **3V3**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
3. Wire the SSD1306 OLED: **VCC** to **3V3**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
4. Copy the project code and paste it into the editor.
5. Click **Run**.
6. Tilt the MPU-6050 sensor back and forth. Watch the difference between the rapidly fluctuating raw values and the smooth filtered output in the Serial Monitor / Serial Plotter.

## Expected Output
Serial Monitor:
```
Raw:1.450,Filtered:1.020
Raw:1.890,Filtered:1.320
Raw:0.910,Filtered:1.180
Raw:1.120,Filtered:1.160
```

## Expected Canvas Behavior
* On execution, the OLED display shows a title line `KALMAN SMOOTHING`.
* As the MPU-6050 tilt changes, the `Raw X` value updates immediately with sensor noise, while the `Filt X` value moves smoothly without spikes.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `P = P + Q` | Predicts how much the estimation error grows between cycles (Process Noise). |
| `K = P / (P + R)` | Calculates the Kalman Gain, defining the weight split between prediction and new measurement. |
| `x_est = x_est + ...` | Adjusts the previous estimate closer to the new reading based on Kalman Gain. |
| `P = (1.0 - K) * P` | Updates the state error covariance for the next step. |
| `display.print(x_est, 3)` | Formats the output to three decimal places. |

## Hardware & Safety Concept
* **Process vs Measurement Noise**: Choosing `Q` (Process noise) and `R` (Measurement noise) is called "tuning". A larger `R` makes the filter trust the prediction more (slower, smoother response). A larger `Q` makes the filter trust the sensor measurements more (faster response, but noisier).
* **I2C Signal Integrity**: Placing multiple sensors on the I2C bus requires matching pull-up resistors (usually 4.7 kΩ). The internal SoC pull-ups might not support high-speed I2C over long wires.

## Try This! (Challenges)
1. **Filter Gyro**: Change the measurement input to the Gyroscope Z-axis angular velocity (`g.gyro.z`) and observe the filtering of spin rates.
2. **Noise Tuner**: Increase the sensor noise covariance `R = 5.0` and observe the extreme delay (lag) introduced to the filtered output.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| OLED doesn't display anything | Wrong address or allocation failure | SSD1306 modules can be set to address `0x3D` instead of `0x3C`. Toggle the address definition. |
| Filtered values are frozen | Kalman gain went to zero | If `P` decays completely due to math errors or bad initialization, reset `P = 1.0` in code if it drops below a threshold. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [60 - OLED SSD1306 Display Setup (I2C)](../intermediate/60-oled-ssd1306-display-setup.md)
- [155 - OLED Complementary Filter Balancer](155-oled-complementary-filter.md)
