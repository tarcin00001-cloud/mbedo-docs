# 155 - ESP32 OLED Complementary Filter Balancer

Build a digital spirit level and balance indicator that implements a Complementary Filter to fuse accelerometer and gyroscope data from an MPU-6050 sensor, rendering a moving bubble on an OLED and sounding a proportional pitch buzzer alarm when tilted.

## Goal
Learn how to implement a Complementary Filter sensor fusion algorithm, understand the balance of high-pass and low-pass filters, and build a proportional acoustic tilt alarm.

## What You Will Build
An MPU-6050 and an SSD1306 OLED are connected to the I2C bus (GPIO 21/22). A buzzer is on GPIO 15. The ESP32 calculates raw accelerometer tilt and integrates gyroscope rates. A Complementary Filter fuses these values. The OLED displays a bubble that moves off-center when tilted. The buzzer beeps faster and at a higher pitch as the tilt angle increases.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MPU-6050 6-DOF IMU Sensor | `mpu6050` | Yes | Yes |
| 0.96″ SSD1306 OLED (I2C) | `oled` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C shared lines |
| MPU-6050 | VCC / GND | 3V3 / GND | Red / Black | IMU Power |
| SSD1306 OLED | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C shared lines |
| SSD1306 OLED | VCC / GND | 3V3 / GND | Red / Black | OLED Power |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Proportional alarm output |
| Active Buzzer | GND (−) | GND | Black | Ground reference |

> **Wiring tip:** Share the I2C bus lines. Connect the active buzzer to GPIO 15. Keep all grounds connected.

## Code
```cpp
// OLED Complementary Filter Balancer (MPU-6050 + OLED + Buzzer)
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

const int BUZZER_PIN = 15;

// Complementary Filter Constants
// Alpha scales how much we trust the gyroscope integration (high-pass filter)
// vs the raw accelerometer angle (low-pass filter)
const float ALPHA = 0.96; 

float compAngle = 0.0; // The calculated tilt angle
unsigned long lastTime = 0;

void setup() {
  Serial.begin(115200);
  
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED init failed!");
    while(1) {}
  }
  
  if (!mpu.begin()) {
    Serial.println("MPU-6050 init failed!");
    while(1) {}
  }
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 20);
  display.print("Balancer starting...");
  display.display();
  
  // Read initial angle to seed the filter
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);
  compAngle = atan2(a.acceleration.y, a.acceleration.z) * 180.0 / M_PI;
  
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
  
  // 1. Calculate raw angle from Accelerometer
  float accAngle = atan2(a.acceleration.y, a.acceleration.z) * 180.0 / M_PI;
  
  // 2. Read gyro angular velocity (Convert rad/s to deg/s)
  float gyroRate = g.gyro.x * 180.0 / M_PI;
  
  // 3. Apply Complementary Filter Formula
  // compAngle = Alpha * (gyro integration) + (1 - Alpha) * (acc angle)
  compAngle = ALPHA * (compAngle + gyroRate * dt) + (1.0 - ALPHA) * accAngle;
  
  // 4. Update OLED Spirit Level HUD
  display.clearDisplay();
  
  // Draw bubble level container (outer box)
  int boxX = 14, boxY = 40, boxW = 100, boxH = 16;
  display.drawRect(boxX, boxY, boxW, boxH, SSD1306_WHITE);
  
  // Draw center lines
  display.drawFastVLine(64, boxY, boxH, SSD1306_WHITE);
  
  // Map tilt angle (-45 to 45 degrees) to bubble position (0 to 100 pixels inside box)
  int bubbleX = map((int)compAngle, -45, 45, 0, boxW);
  bubbleX = constrain(bubbleX, 4, boxW - 12); // Keep inside box borders
  
  // Draw the bubble (filled circle)
  display.fillCircle(boxX + bubbleX + 4, boxY + 8, 5, SSD1306_WHITE);
  
  // Display text values
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.print("SPIRIT LEVEL HUD");
  display.drawFastHLine(0, 10, 128, SSD1306_WHITE);
  
  display.setCursor(0, 15);
  display.print("Angle: "); display.print(compAngle, 1); display.print(" deg");
  display.setCursor(0, 25);
  display.print("Raw:   "); display.print(accAngle, 1);  display.print(" deg");
  
  display.display();
  
  // 5. Sound Proportional Alarm
  // If tilt exceeds 5 degrees, beep faster as tilt increases
  float absAngle = abs(compAngle);
  if (absAngle > 5.0) {
    // Map angle (5 to 45 degrees) to beep delay (500ms down to 50ms)
    int beepDelay = map((int)absAngle, 5, 45, 500, 50);
    beepDelay = constrain(beepDelay, 50, 500);
    
    digitalWrite(BUZZER_PIN, HIGH);
    delay(30);
    digitalWrite(BUZZER_PIN, LOW);
    
    // Non-blocking sleep equivalent (since buzzer delay is small, 30ms)
    // In a physical loop, avoid delay, but this is simple proportional warning
    delay(beepDelay);
  } else {
    digitalWrite(BUZZER_PIN, LOW);
    delay(100); // 10Hz poll rate when balanced
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MPU-6050**, **SSD1306 OLED**, and **Buzzer** onto the canvas.
2. Wire I2C shared lines to **GPIO21/GPIO22** and Buzzer to **GPIO15**.
3. Paste the code and click **Run**.
4. Slide the MPU-6050 Y acceleration slider (tilting the sensor). Watch the bubble circle move off-center on the OLED.
5. If the slider tilts past 5 degrees, watch the buzzer widget flash green (beeping) faster as the tilt increases.

## Expected Output
Serial Monitor:
```
Angle: 0.2 | Raw: 0.2
Angle: 15.4 | Raw: 16.2 (Buzzer beeps slow)
Angle: 32.1 | Raw: 34.0 (Buzzer beeps fast)
```

OLED Display Layout:
```
SPIRIT LEVEL HUD
─────────────────
Angle: 15.4 deg
Raw:   16.2 deg
 ┌──────┬──────┐
 │      ●      │  (Bubble circle shifted off-center)
 └──────┴──────┘
```

## Expected Canvas Behavior
* At boot, LCD shows the bubble level centered.
* Dragging the IMU Y-acceleration slider shifts the bubble circle left or right.
* If the bubble shifts past the center indicator lines, the buzzer widget pulses.
* The beeping speed increases as the slider is dragged further from 0.

## Code Walkthrough
| Line | Math / Step |
| --- | --- |
| `compAngle = ALPHA * (compAngle + ...) + (1.0 - ALPHA) * accAngle` | **Complementary Filter**: Combines high-pass filtered gyro integration with low-pass filtered accelerometer angles. |
| `map((int)compAngle, -45, 45, 0, boxW)` | Maps the angle to horizontal pixel coordinates inside the box container. |
| `map((int)absAngle, 5, 45, 500, 50)` | Scales the beep interval proportionally to the tilt angle. |

## Hardware & Safety Concept: Sensor Fusion and Complementary Filters
A **Complementary Filter** is a simple and computationally efficient alternative to a Kalman Filter. It combines two sensor inputs using a weighted sum:
\[\text{Angle} = \alpha \cdot (\text{Angle} + \text{GyroRate} \cdot dt) + (1 - \alpha) \cdot \text{AccAngle}\]
The term \(\alpha\) (typically 0.96 to 0.98) functions as a **high-pass filter** for the gyroscope (trusting its fast changes over short time frames) and a **low-pass filter** for the accelerometer (trusting its absolute gravity alignment over long time frames). This filters out both high-frequency vibration noise and low-frequency drift.

## Try This! (Challenges)
1. **Dual Axis bubble level**: Expand the graph to draw a 2D crosshair bubble level, monitoring both Pitch and Roll axes.
2. **Dynamic Alpha Tuner**: Connect a potentiometer (GPIO 34) to dynamically adjust `ALPHA` from 0.80 to 0.99.
3. **Auto Calibration home**: Read and store sensor offsets on startup if a button is held down.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Bubble moves in opposite direction of tilt | Mapping coordinates inverted | Swap the low/high values in the `map()` function (e.g. swap 0 and `boxW`) |
| Buzzer does not sound | Buzzer pin misconfigured | Confirm active buzzer is wired to GPIO 15 |
| Bubble level oscillates wildly | `ALPHA` constant is too low | Increase `ALPHA` to 0.98 to rely more on gyroscope smoothing |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [94 - ESP32 MPU-6050 Gyroscope Position HUD](../intermediate/94-esp32-mpu6050-gyroscope-position-hud.md)
- [154 - ESP32 OLED Kalman Filter Demo](154-esp32-oled-kalman-filter-demo.md)
- [152 - ESP32 Servo PID Angle Balancer](152-esp32-servo-pid-angle-balancer.md)
