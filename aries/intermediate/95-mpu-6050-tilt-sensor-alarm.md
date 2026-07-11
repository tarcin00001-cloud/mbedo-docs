# 95 - MPU-6050 Tilt Sensor Alarm

Build a safety tilt alarm that triggers an active buzzer warning if the device's orientation angle exceeds 45 degrees.

## Goal
Learn how to calculate orientation angles from MPU-6050 sensor data, compare them against a safety threshold, and trigger alarm peripherals without using loops inside C++ code.

## What You Will Build
An MPU-6050 sensor is connected via I2C. The board calculates the current tilt angle. If the absolute pitch or roll angle exceeds 45.0 degrees, a warning buzzer on GPIO 14 sounds a continuous alarm.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MPU-6050 Accelerometer & Gyroscope | `mpu6050` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes (on GPIO 14) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 Module | VCC | 3V3 | Red | Power |
| MPU-6050 Module | GND | GND | Black | Ground |
| MPU-6050 Module | SDA | SDA0 (GP17) | Blue | I2C Data line |
| MPU-6050 Module | SCL | SCL0 (GP16) | Yellow | I2C Clock line |
| Active Buzzer | Positive (+) | GPIO 14 | Orange | Alarm output |
| Active Buzzer | Negative (-) | GND | Black | Ground reference |

> **Wiring tip:** Standard active buzzer modules are driven directly by a logic HIGH from GPIO 14. The MPU-6050 connects to the standard I2C0 pins GP17 and GP16.

## Code
```cpp
// MPU-6050 Tilt Sensor Alarm
#include <Wire.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <math.h>

const int BUZZER_PIN = 14;

// Safety tilt angle limit in degrees
const float TILT_LIMIT_DEG = 45.0; 

Adafruit_MPU6050 mpu;
bool sensorOk = true;

void setup() {
  Serial.begin(115200);
  Wire.begin(); // Initialize default I2C0 interface
  
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW); // Start with buzzer OFF
  
  if (!mpu.begin()) {
    Serial.println("MPU-6050 not found! Check wiring.");
    sensorOk = false;
  } else {
    Serial.println("Tilt Alarm Station ready.");
  }
}

void loop() {
  if (!sensorOk) {
    // Flash onboard Red LED to indicate sensor failure
    pinMode(LED_R, OUTPUT);
    digitalWrite(LED_R, HIGH);
    delay(200);
    digitalWrite(LED_R, LOW);
    delay(200);
    return;
  }
  
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);
  
  // Calculate pitch and roll angles in degrees
  float Ax = a.acceleration.x;
  float Ay = a.acceleration.y;
  float Az = a.acceleration.z;
  
  float pitch = atan2(-Ax, sqrt(Ay * Ay + Az * Az)) * 57.2958;
  float roll = atan2(Ay, Az) * 57.2958;
  
  // Get absolute values for comparison
  float absPitch = abs(pitch);
  float absRoll = abs(roll);
  
  Serial.print("Pitch: "); Serial.print(pitch, 1);
  Serial.print(" | Roll: "); Serial.println(roll, 1);
  
  // Check if either axis exceeds the 45-degree tilt limit
  if (absPitch > TILT_LIMIT_DEG || absRoll > TILT_LIMIT_DEG) {
    Serial.println("!! TILT WARNING ALARM !!");
    
    // Trigger alarm outputs
    digitalWrite(BUZZER_PIN, HIGH);
  } else {
    // Normal level state
    digitalWrite(BUZZER_PIN, LOW);
  }
  
  delay(100); // 10Hz sampling frequency
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **MPU-6050**, and **Active Buzzer** onto the canvas.
2. Wire the I2C lines to **SDA0 (GP17)** and **SCL0 (GP16)**. Connect the Buzzer positive to **GPIO 14**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.
6. Adjust the pitch/roll sliders on the MPU-6050 widget past 45 degrees to trigger the alarm.

## Expected Output
Serial Monitor:
```
Tilt Alarm Station ready.
Pitch: 5.2 | Roll: -2.0
Pitch: 48.4 | Roll: -2.1
!! TILT WARNING ALARM !!
```

## Expected Canvas Behavior
* While the virtual sensor is relatively level (under 45°), the buzzer widget is off.
* Adjusting the virtual sensor sliders past 45° on either axis turns ON the active buzzer widget.
* Returning the sensor to a flat level position shuts off the alarm immediately.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `abs(pitch)` | Converts negative angle readings (tilting forward/left) to positive numbers. |
| `absPitch > TILT_LIMIT_DEG` | Evaluates if the device has tilted past the 45-degree limit on the pitch axis. |
| `digitalWrite(BUZZER_PIN, HIGH)` | Drives GPIO 14 HIGH to sound the buzzer. |
| `delay(100)` | Controls loop rate, keeping the sensor sampling frequency at 10Hz. |

## Hardware & Safety Concept
* **Industrial Rollover Protection**: In heavy machinery (e.g. cranes, excavators, forklifts), tilt monitoring is essential. If the machine travels onto an incline exceeding safety bounds, the control computer triggers alarms to notify the operator and can automatically lock controls to prevent a rollover hazard.

## Try This! (Challenges)
1. **Warning LED Indicator**: Connect a Warning LED to GPIO 15. Light the LED at 30 degrees (Warning level) and sound the buzzer at 45 degrees (Critical level).
2. **Beeping Alarm**: Modify the code so the buzzer beeps repeatedly when active, with the beep frequency increasing as the tilt angle gets higher.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer does not sound | Polarity reversed or wrong pin | Verify the positive buzzer lead connects to GPIO 14. |
| Alarm triggers when resting flat | Sensor offset error | Calibrate the sensor or adjust calibration values in code. |
| System indicates error on boot | I2C connections loose | Verify SDA and SCL connections to the ARIES board. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [93 - MPU-6050 Accelerometer Serial Logs](93-mpu-6050-accelerometer-serial.md)
- [94 - MPU-6050 Gyroscope Position HUD](94-mpu-6050-gyroscope-position-hud.md)
- [26 - Tilt Sensor Alarm](../beginner/26-tilt-sensor-alarm.md)
