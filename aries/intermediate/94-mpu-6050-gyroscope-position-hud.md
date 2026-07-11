# 94 - MPU-6050 Gyroscope Position HUD

Read orientation data from an MPU-6050 sensor and display live Pitch and Roll angles on an SSD1306 OLED screen connected to the ARIES v3 board.

## Goal
Learn how to calculate pitch and roll orientation angles using gravity vector components from the accelerometer, and update a graphic OLED HUD without using loops inside C++ code.

## What You Will Build
An MPU-6050 sensor reads orientation data, and an SSD1306 OLED screen displays the calculated Pitch and Roll angles in degrees. The display updates every 200 ms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MPU-6050 Accelerometer & Gyroscope | `mpu6050` | Yes | Yes |
| 0.96" SSD1306 OLED Display | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 Module | VCC | 3V3 | Red | Power |
| MPU-6050 Module | GND | GND | Black | Ground |
| MPU-6050 Module | SDA | SDA0 (GP17) | Blue | I2C Data (shared) |
| MPU-6050 Module | SCL | SCL0 (GP16) | Yellow | I2C Clock (shared) |
| SSD1306 OLED | VCC | 3V3 | Red | OLED power supply (3.3V) |
| SSD1306 OLED | GND | GND | Black | Ground reference |
| SSD1306 OLED | SDA | SDA0 (GP17) | Blue | I2C Data (shared) |
| SSD1306 OLED | SCL | SCL0 (GP16) | Yellow | I2C Clock (shared) |

> **Wiring tip:** Share the I2C0 bus lines SDA0 (GP17) and SCL0 (GP16) in parallel on the breadboard for both the MPU-6050 and the OLED display.

## Code
```cpp
// MPU-6050 Gyroscope Position HUD
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
bool sensorOk = true;

void setup() {
  Serial.begin(115200);
  Wire.begin(); // Initialize default I2C0 interface
  
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED init failed.");
    sensorOk = false;
  }
  
  if (!mpu.begin()) {
    Serial.println("MPU-6050 init failed.");
    sensorOk = false;
  }
  
  if (sensorOk) {
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 0);
    display.print("MPU6050 HUD Ready");
    display.display();
    delay(1000);
  }
}

void loop() {
  if (!sensorOk) {
    delay(1000);
    return;
  }
  
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
  
  // Update OLED display
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  
  display.setCursor(0, 0);
  display.print("--- ORIENTATION ---");
  
  display.setCursor(0, 20);
  display.print("Pitch: ");
  display.print(pitch, 1);
  display.print(" deg");
  
  display.setCursor(0, 40);
  display.print("Roll:  ");
  display.print(roll, 1);
  display.print(" deg");
  
  display.display(); // Push buffer to screen
  
  Serial.print("Pitch: "); Serial.print(pitch, 1);
  Serial.print(" | Roll: "); Serial.println(roll, 1);
  
  delay(200); // 5Hz update rate
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **MPU-6050**, and **SSD1306 OLED** components onto the canvas.
2. Wire both SDA pins to **SDA0 (GP17)** and SCL pins to **SCL0 (GP16)**. Connect VCC to **3V3**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.
6. Adjust the rotational pitch/roll sliders on the MPU-6050 component widget and observe the angles update on the OLED screen.

## Expected Output
Serial Monitor:
```
Pitch: 0.0 | Roll: 0.0
Pitch: 15.4 | Roll: -2.3
Pitch: -45.1 | Roll: 12.0
```

OLED Screen (resting at minor tilt):
```
--- ORIENTATION ---
Pitch: 15.4 deg
Roll:  -2.3 deg
```

## Expected Canvas Behavior
* The OLED display updates orientation angles in real-time.
* Rotating or tilting the virtual sensor widget instantly changes the displayed angle values on the OLED.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `display.begin(...)` | Initialises the OLED graphic screen controller. |
| `atan2(...)` | Trigonometric function used to calculate angles over a full 360-degree range. |
| `57.2958` | Conversion factor to translate Radians into Degrees. |
| `display.clearDisplay()` | Wipes the local graphic buffer before painting new text. |
| `display.display()` | Transmits the updated graphic buffer over the shared I2C bus. |

## Hardware & Safety Concept
* **Attitude and Heading Reference Systems (AHRS)**: Calculating pitch and roll using acceleration gravity vectors works well when the sensor is stationary (static orientation). In dynamic environments (e.g. flight controllers on drones), physical movement forces create noise on the acceleration data. In those systems, developers combine the accelerometer with gyroscope angular velocity readings using a **Complementary Filter** or **Kalman Filter** to get clean, stable orientation estimates.

## Try This! (Challenges)
1. **Horizon Indicator Line**: Draw a simple horizontal reference line across the center of the OLED screen that tilts at the calculated roll angle.
2. **Dynamic Calibration**: Implement an auto-zero button on GPIO 16 that offsets the current orientation to 0 degrees when clicked.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Angle numbers jitter rapidly | Sensor noise | Implement a simple software low-pass filter on the pitch/roll variables. |
| Screen remains black | OLED or MPU not starting | Verify SDA/SCL connections and ensure I2C address is correct. |
| Values are inverted | Axis orientations swapped | Modify the sign (plus/minus) of Ax, Ay, or Az in the formulas. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [93 - MPU-6050 Accelerometer Serial Logs](93-mpu-6050-accelerometer-serial.md)
- [95 - MPU-6050 Tilt Sensor Alarm](95-mpu-6050-tilt-sensor-alarm.md)
- [60 - OLED SSD1306 Display Setup (I2C)](60-oled-ssd1306-display-setup.md)
