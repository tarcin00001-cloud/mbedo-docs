# 185 - ESP32 Complex PID Balancer

Build a dual-axis closed-loop self-balancing platform that measures Pitch and Roll tilt angles using an MPU-6050 sensor, processes them through two independent PID controllers, drives two servos (X and Y axes) to stabilize the platform, and renders a 2D crosshair target on an OLED display.

## Goal
Learn how to implement multiple parallel PID control loops, map multi-axis angular errors to servo corrections, and draw 2D crosshair target coordinate graphics.

## What You Will Build
An MPU-6050 IMU and an SSD1306 OLED are connected to the I2C bus (GPIO 21/22). Two servos (X-Axis: GPIO 12, Y-Axis: GPIO 13) steer the balancing platform. The ESP32 calculates pitch and roll, runs two separate PID loops, and moves the servos to keep the platform level (0° Pitch, 0° Roll) while plotting a bullseye target showing the platform's balance.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MPU-6050 6-DOF IMU Sensor | `mpu6050` | Yes | Yes |
| Servo Motors (2) | `servo` | Yes | Yes |
| 0.96″ SSD1306 OLED (I2C) | `oled` | Yes | Yes |
| External Power Supply (5V, 2A) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 Sensor | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus (Shared) |
| SSD1306 OLED | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus (Shared) |
| Servo X (Pitch) | Signal | GPIO12 | Orange | Pitch actuator control |
| Servo Y (Roll) | Signal | GPIO13 | Blue | Roll actuator control |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the I2C bus. Power both servos from an external 5V power supply, sharing the ground with the ESP32 to prevent brownouts.

## Code
```cpp
// Complex PID Balancer (Dual-axis MPU-6050 + 2x Servo + OLED)
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <ESP32Servo.h>
#include <math.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);
Adafruit_MPU6050 mpu;

Servo servoX; // Controls Pitch
Servo servoY; // Controls Roll

const int SERVO_X_PIN = 12;
const int SERVO_Y_PIN = 13;

// PID Tuning Constants
const float KP = 1.6;
const float KI = 0.15;
const float KD = 0.08;

// Target angles (0 degrees = level)
const float TARGET_X = 0.0;
const float TARGET_Y = 0.0;

// Servos center offset
const int SERVO_CENTER = 90;

// PID Struct to track states for each axis
struct PIDController {
  float error = 0.0;
  float lastError = 0.0;
  float integral = 0.0;
  float derivative = 0.0;
  
  float compute(float target, float current, float dt) {
    error = target - current;
    
    // Integral with windup clamping
    integral += error * dt;
    integral = constrain(integral, -25, 25);
    
    // Derivative
    derivative = (error - lastError) / dt;
    lastError = error;
    
    return (KP * error) + (KI * integral) + (KD * derivative);
  }
};

PIDController pidX;
PIDController pidY;

unsigned long lastTime = 0;

void setup() {
  Serial.begin(115200);
  
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED allocation failed!");
    while(1) {}
  }
  
  if (!mpu.begin()) {
    Serial.println("MPU-6050 init failed!");
    while(1) {}
  }
  
  mpu.setAccelerometerRange(MPU6050_RANGE_4_G);
  
  servoX.attach(SERVO_X_PIN);
  servoY.attach(SERVO_Y_PIN);
  
  servoX.write(SERVO_CENTER);
  servoY.write(SERVO_CENTER);
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(15, 20);
  display.print("PID Dual Balancer");
  display.setCursor(15, 35);
  display.print("Initializing...");
  display.display();
  
  lastTime = millis();
  delay(1500);
}

void loop() {
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);
  
  unsigned long now = millis();
  float dt = (now - lastTime) / 1000.0;
  if (dt <= 0.0) dt = 0.01;
  lastTime = now;
  
  // 1. Calculate Pitch and Roll angles from accelerometer data
  float pitch = atan2(-a.acceleration.x, sqrt(a.acceleration.y * a.acceleration.y + a.acceleration.z * a.acceleration.z)) * 180.0 / M_PI;
  float roll  = atan2(a.acceleration.y, a.acceleration.z) * 180.0 / M_PI;
  
  // 2. Compute PID corrections
  float corrX = pidX.compute(TARGET_X, pitch, dt);
  float corrY = pidY.compute(TARGET_Y, roll, dt);
  
  // Calculate target servo angles
  int angleX = SERVO_CENTER + (int)corrX;
  int angleY = SERVO_CENTER + (int)corrY;
  
  // Constrain servo limits to prevent joint binding
  angleX = constrain(angleX, 35, 145);
  angleY = constrain(angleY, 35, 145);
  
  // 3. Command Servos
  servoX.write(angleX);
  servoY.write(angleY);
  
  // 4. Update OLED 2D Crosshair Target UI
  display.clearDisplay();
  
  // Draw outer target circle (centered at 96, 32 with radius 20)
  int cx = 96;
  int cy = 32;
  display.drawCircle(cx, cy, 20, SSD1306_WHITE);
  display.drawCircle(cx, cy, 10, SSD1306_WHITE);
  display.drawFastHLine(cx - 24, cy, 48, SSD1306_WHITE);
  display.drawFastVLine(cx, cy - 24, 48, SSD1306_WHITE);
  
  // Map Pitch and Roll angles to 2D target offset coordinates
  // Scale so 30 degrees = 20 pixels offset
  int dotX = cx + map((int)roll, -30, 30, -20, 20);
  int dotY = cy + map((int)pitch, -30, 30, 20, -20); // Inverted Y-axis
  
  dotX = constrain(dotX, cx - 20, cx + 20);
  dotY = constrain(dotY, cy - 20, cy + 20);
  
  // Draw current position dot (platform balance indicator)
  display.fillCircle(dotX, dotY, 3, SSD1306_WHITE);
  
  // Print textual HUD
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.print("PID BALANCER");
  display.drawFastHLine(0, 10, 64, SSD1306_WHITE);
  
  display.setCursor(0, 15);
  display.print("P: "); display.print(pitch, 1); display.print(" deg");
  display.setCursor(0, 28);
  display.print("R: "); display.print(roll, 1); display.print(" deg");
  
  display.setCursor(0, 45);
  display.print("SX: "); display.print(angleX);
  display.setCursor(0, 55);
  display.print("SY: "); display.print(angleY);
  
  display.display();
  
  delay(15); // ~66Hz refresh rate
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MPU-6050**, **SSD1306 OLED**, and two **Servos** onto the canvas.
2. Wire shared I2C bus to **GPIO21/GPIO22**, Servo X to **GPIO12**, and Servo Y to **GPIO13**.
3. Paste the code and click **Run**.
4. Slide the MPU-6050 X and Y acceleration vectors. Watch the two servos rotate to compensate.
5. Watch the dot on the OLED target circle shift in 2D space to track the platform's balance.

## Expected Output
Serial Monitor:
* Real-time dual-axis PID calculations and servo updates.

OLED Layout:
```
PID BALANCER       │
P: 12.4 deg     ───●─── (OLED target crosshair with balance dot)
R: -8.5 deg        │
SX: 110 SY: 76
```

## Expected Canvas Behavior
* At boot, the OLED target shows the dot centered. Both servos are at 90°.
* Tilting the IMU X-acceleration slider moves Servo X and shifts the target dot vertically.
* Tilting the Y-acceleration slider moves Servo Y and shifts the target dot horizontally.
* The servos move dynamically to return the platform to center.

## Code Walkthrough
| Line | Math / Step |
| --- | --- |
| `pitch = atan2(-ax, ...)` | Calculates the pitch angle in degrees. |
| `roll = atan2(ay, az) ...` | Calculates the roll angle in degrees. |
| `pidX.compute(...)` | Calculates the PID correction for the Pitch axis. |
| `dotX = cx + map(...)` | Maps the roll angle to the horizontal position of the target dot. |

## Hardware & Safety Concept: Multi-Axis Closed-loop Control Systems
Self-balancing platforms (like camera gimbals or segways) use multi-axis closed-loop control systems. These systems run independent PID loops for each axis. The loops run in parallel:
1. **Sensors**: The IMU measures the orientation angles (pitch, roll).
2. **Controller**: The PID loops calculate the correction needed for each axis.
3. **Actuators**: The servos adjust the angles to keep the platform level.
This feedback loop runs continuously to maintain stability.

## Try This! (Challenges)
1. **Dynamic Target Shift**: Add a joystick (Project 97) to allow manually steering the target balance center.
2. **Calibration home**: Save the platform's flat reference offsets to EEPROM (Project 182) on button press.
3. **Emergency Disconnect**: Cut off the servos if the tilt exceeds 60 degrees to prevent mechanical damage.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servos rotate in the same direction as the tilt (amplifies error) | Correction sign inverted | Change the sign of the correction value in the servo write calls |
| The platform vibrates and shakes wildly | PID constants are too high | Reduce the proportional gain `KP` and derivative gain `KD` constants |
| One axis does not respond | Pin mapping conflict | Verify that Servo X is connected to GPIO 12 and Servo Y to GPIO 13 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [152 - ESP32 Servo PID Angle Balancer](152-esp32-servo-pid-angle-balancer.md)
- [183 - ESP32 Dual-core Processing Balancer](183-esp32-dual-core-processing-balancer.md)
- [94 - ESP32 MPU-6050 Gyroscope Position HUD](../intermediate/94-esp32-mpu6050-gyroscope-position-hud.md)
