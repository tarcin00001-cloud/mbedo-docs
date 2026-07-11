# 154 - ESP32 OLED Kalman Filter Demo

Build a real-time signal filtering visualizer that reads raw accelerometer and gyroscope data from an MPU-6050 sensor, applies a 1D Kalman filter to combine the data into a stable angle, and plots raw vs. filtered graphs on an OLED screen.

## Goal
Learn how to implement a 1D Kalman filter algorithm, combine accelerometer angles and gyroscope angular rates, and plot real-time scrolling waveforms on an SSD1306 OLED.

## What You Will Build
An MPU-6050 IMU and an SSD1306 OLED are connected to the I2C bus (GPIO 21/22). The ESP32 calculates raw tilt angles from the accelerometer, and rate of change from the gyroscope. The Kalman filter combines them to estimate a clean tilt angle. The OLED displays a scrolling graph of both the raw (noisy) and Kalman-filtered (smooth) angles.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MPU-6050 6-DOF IMU Sensor | `mpu6050` | Yes | Yes |
| 0.96″ SSD1306 OLED (I2C) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C shared lines |
| MPU-6050 | VCC / GND | 3V3 / GND | Red / Black | IMU Power |
| SSD1306 OLED | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C shared lines |
| SSD1306 OLED | VCC / GND | 3V3 / GND | Red / Black | OLED Power |

> **Wiring tip:** The I2C bus is shared. The HMC5883L is not used in this project, but if it is wired, it can remain on the bus.

## Code
```cpp
// OLED Kalman Filter Demo (MPU-6050 + OLED Graph)
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <math.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);
Adafruit_MPU6050 mpu;

// 1D Kalman Filter Struct
struct Kalman1D {
  // Model parameters: Process noise covariance (Q) and Measurement noise covariance (R)
  float Q_angle = 0.001; // Process noise covariance for the accelerometer angle
  float Q_bias  = 0.003; // Process noise covariance for the gyroscope bias
  float R_measure = 0.03; // Measurement noise covariance

  float angle = 0.0; // The calculated angle (State estimate)
  float bias = 0.0;  // The calculated gyro bias (State estimate)
  
  // Error covariance matrix (2x2) P
  float P[2][2] = {
    { 0.0, 0.0 },
    { 0.0, 0.0 }
  };

  // Update estimate with new measurements
  // newAngle: raw angle from accelerometer (degrees)
  // newRate: raw angular velocity from gyroscope (degrees/second)
  // dt: time step (seconds)
  float getFilteredAngle(float newAngle, float newRate, float dt) {
    // 1. Predict state ahead
    float rate = newRate - bias;
    angle += dt * rate;

    // 2. Predict error covariance ahead
    P[0][0] += dt * (dt * P[1][1] - P[0][1] - P[1][0] + Q_angle);
    P[0][1] -= dt * P[1][1];
    P[1][0] -= dt * P[1][1];
    P[1][1] += Q_bias * dt;

    // 3. Calculate Kalman Gain
    float S = P[0][0] + R_measure;
    float K[2]; // Kalman gain (2x1 vector)
    K[0] = P[0][0] / S;
    K[1] = P[1][0] / S;

    // 4. Correct estimate with measurement (Update)
    float y = newAngle - angle; // Measurement residual
    angle += K[0] * y;
    bias  += K[1] * y;

    // 5. Update error covariance
    float P00_temp = P[0][0];
    float P01_temp = P[0][1];

    P[0][0] -= K[0] * P00_temp;
    P[0][1] -= K[0] * P01_temp;
    P[1][0] -= K[1] * P00_temp;
    P[1][1] -= K[1] * P01_temp;

    return angle;
  }
};

Kalman1D kalman;

// Scrolling History Buffers (OLED Graph coordinates)
const int GRAPH_WIDTH = 128;
const int GRAPH_HEIGHT = 40;
const int GRAPH_Y_OFFSET = 24;

int rawHistory[GRAPH_WIDTH];
int kalmanHistory[GRAPH_WIDTH];
int writeIdx = 0;

unsigned long lastTime = 0;

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
  
  // Clear buffers
  for (int i = 0; i < GRAPH_WIDTH; i++) {
    rawHistory[i] = GRAPH_HEIGHT / 2;
    kalmanHistory[i] = GRAPH_HEIGHT / 2;
  }
  
  lastTime = millis();
}

void loop() {
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);
  
  unsigned long now = millis();
  float dt = (now - lastTime) / 1000.0;
  if (dt <= 0.0) dt = 0.01;
  lastTime = now;
  
  // 1. Calculate raw angle from Accelerometer
  // Roll = atan2(ay, az)
  float rawAngle = atan2(a.acceleration.y, a.acceleration.z) * 180.0 / M_PI;
  
  // 2. Read gyro angular velocity (Convert rad/s to deg/s)
  float gyroRate = g.gyro.x * 180.0 / M_PI;
  
  // 3. Filter raw angle with Kalman Filter
  float filteredAngle = kalman.getFilteredAngle(rawAngle, gyroRate, dt);
  
  // Map angles (-60 to 60 degrees) to graph height (0 to GRAPH_HEIGHT)
  // Inverse mapping so positive tilt is up on screen
  int rawMap = map((int)rawAngle, -60, 60, GRAPH_HEIGHT, 0);
  int kalmanMap = map((int)filteredAngle, -60, 60, GRAPH_HEIGHT, 0);
  
  rawMap = constrain(rawMap, 0, GRAPH_HEIGHT);
  kalmanMap = constrain(kalmanMap, 0, GRAPH_HEIGHT);
  
  // Save to history buffers (Circular buffers)
  rawHistory[writeIdx] = rawMap;
  kalmanHistory[writeIdx] = kalmanMap;
  writeIdx = (writeIdx + 1) % GRAPH_WIDTH;
  
  // Draw Graph
  display.clearDisplay();
  
  // Header text HUD
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.print("RAW (dotted) vs KALMAN");
  display.setCursor(0, 10);
  display.print("Angle: "); display.print(filteredAngle, 1); display.print(" deg");
  
  // Plot histories
  for (int x = 0; x < GRAPH_WIDTH; x++) {
    // Map circular index
    int readIdx = (writeIdx + x) % GRAPH_WIDTH;
    
    int yRaw = rawHistory[readIdx] + GRAPH_Y_OFFSET;
    int yKalman = kalmanHistory[readIdx] + GRAPH_Y_OFFSET;
    
    // Draw raw data as small dotted points
    if (x % 2 == 0) {
      display.drawPixel(x, yRaw, SSD1306_WHITE);
    }
    
    // Draw Kalman filtered data as a solid line
    if (x > 0) {
      int prevIdx = (writeIdx + x - 1) % GRAPH_WIDTH;
      int yPrevKalman = kalmanHistory[prevIdx] + GRAPH_Y_OFFSET;
      display.drawLine(x - 1, yPrevKalman, x, yKalman, SSD1306_WHITE);
    }
  }
  
  display.display();
  
  Serial.print("Raw: "); Serial.print(rawAngle);
  Serial.print(" | Kalman: "); Serial.println(filteredAngle);
  
  delay(30); // ~33Hz refresh rate
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MPU-6050**, and **SSD1306 OLED** onto the canvas.
2. Wire both I2C lines to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the MPU-6050 Y acceleration slider (tilting the sensor).
5. Watch the OLED screen plot a noisy dotted line (raw) and a smooth solid line (Kalman-filtered) following the tilt.

## Expected Output
Serial Monitor:
```
Raw: 12.4 | Kalman: 12.1
Raw: 14.8 | Kalman: 13.5
Raw: 11.2 | Kalman: 12.8
```

OLED Screen:
```
RAW (dotted) vs KALMAN
Angle: 12.8 deg
~~~~~~~~~~~~~~~~~~~~~~ (Solid line representing smooth Kalman data)
. . .  . .   . .  .    (Dotted points representing noisy raw data)
```

## Expected Canvas Behavior
* The OLED widget prints the current angle.
* Tilting the IMU slider plots two overlapping graphs: a dotted graph (raw accelerometer data) and a smooth solid graph (Kalman filter estimate).
* The Kalman graph tracks the angle changes smoothly without lag.

## Code Walkthrough
| Line | Math / Step |
| --- | --- |
| `angle += dt * rate` | **Predict Step**: Integrates the gyro angular rate to predict the new angle. |
| `K[0] = P[0][0] / S` | **Calculate Gain**: Calculates the Kalman gain representing how much to trust the accelerometer vs the gyroscope. |
| `angle += K[0] * y` | **Correct Step**: Updates the angle estimate with the raw accelerometer measurement. |

## Hardware & Safety Concept: Sensor Fusion and the Kalman Filter
Accelerometers and gyroscopes have opposite error characteristics:
1. **Accelerometers**: Measure gravity vectors to calculate absolute angle, but are highly sensitive to vibration and movement acceleration (high-frequency noise).
2. **Gyroscopes**: Measure angular velocity. Integrating this velocity gives smooth angles, but the integration accumulates tiny sensor offsets over time, causing the angle to drift (low-frequency drift).
A **Kalman Filter** is a mathematical algorithm that combines both sensors in real time (sensor fusion). It uses the gyroscope for smooth, fast updates, and the accelerometer to correct for drift, providing a clean, lag-free estimate of the true angle.

## Try This! (Challenges)
1. **Dynamic Noise Tuner**: Connect a potentiometer (GPIO 34) to dynamically adjust the measurement noise `R_measure` from 0.01 to 0.5, observing the change on the OLED.
2. **Complementary Filter Comparison**: Plot a third line showing a complementary filter output (`angle = 0.98 * (angle + gyro * dt) + 0.02 * accAngle`) to compare algorithms.
3. **Extreme Tilt Alert**: Sound a buzzer (GPIO 4) if the filtered angle exceeds 50 degrees.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Kalman filter output drifts continuously | Gyroscope bias not subtracting | Verify `rate = newRate - bias` is executing in the predict step |
| Filter output lags far behind tilts | `R_measure` is set too high | Lower `R_measure` to make the filter trust the accelerometer measurements more |
| Screen renders slowly | High math load | Minimize OLED drawings; ensure `display.display()` is called only once per loop cycle |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [94 - ESP32 MPU-6050 Gyroscope Position HUD](../intermediate/94-esp32-mpu6050-gyroscope-position-hud.md)
- [149 - ESP32 I2C OLED Tilt-compensated Compass](149-esp32-i2c-oled-tilt-compensated-compass.md)
- [155 - ESP32 OLED Complementary Filter Balancer](155-esp32-oled-complementary-filter-balancer.md) (Next project)
