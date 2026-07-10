# 95 - ESP32 MPU-6050 Tilt Sensor Alarm

Build a digital level or safety tilt alarm that triggers a warning LED and buzzer if the device's orientation angle exceeds a safety threshold.

## Goal
Learn how to monitor pitch and roll calculations, compare orientation angles against threshold parameters, and trigger safety alarm states.

## What You Will Build
An MPU-6050 sensor connected via I2C. The ESP32 calculates the current tilt angle. If the absolute pitch or roll angle exceeds 30 degrees, an alert LED on GPIO 5 turns on and a buzzer on GPIO 15 sounds a continuous alarm.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MPU-6050 Accelerometer & Gyroscope | `mpu6050` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 Module | VCC | 3V3 | Red | Power |
| MPU-6050 Module | GND | GND | Black | Ground |
| MPU- MP6050 | SDA | GPIO21 | Blue | I2C Data line |
| MPU-6050 | SCL | GPIO22 | Yellow | I2C Clock line |
| LED (Red) | Anode (+) | GPIO5 via 330 Ω | Orange | Indicator output |
| LED (Red) | Cathode (−) | GND | Black | Ground reference |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Alarm output |
| Active Buzzer | GND (−) | GND | Black | Ground reference |

> **Wiring tip:** Standard active buzzer modules are driven directly by a logic HIGH from GPIO 15. The MPU-6050 uses the standard I2C connection.

## Code
```cpp
// MPU-6050 Tilt Sensor Alarm
#include <Wire.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <math.h>

const int LED_PIN = 5;
const int BUZZER_PIN = 15;

// Safety tilt angle limit in degrees
const float TILT_LIMIT_DEG = 30.0; 

Adafruit_MPU6050 mpu;

void setup() {
  Serial.begin(115200);
  
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  digitalWrite(LED_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
  
  if (!mpu.begin()) {
    Serial.println("MPU-6050 not found! System halt.");
    while (1) {
      // Fast flash LED to indicate sensor error status
      digitalWrite(LED_PIN, HIGH);
      delay(100);
      digitalWrite(LED_PIN, LOW);
      delay(100);
    }
  }
  
  Serial.println("Tilt Alarm Station ready.");
}

void loop() {
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
  
  // Check if either axis exceeds tilt limit
  if (absPitch > TILT_LIMIT_DEG || absRoll > TILT_LIMIT_DEG) {
    Serial.println("!! TILT WARNING ALARM !!");
    
    // Trigger alarm outputs
    digitalWrite(LED_PIN, HIGH);
    digitalWrite(BUZZER_PIN, HIGH);
  } else {
    // Normal level state
    digitalWrite(LED_PIN, LOW);
    digitalWrite(BUZZER_PIN, LOW);
  }
  
  delay(100); // 10Hz sampling frequency
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MPU-6050**, **LED**, and **Buzzer** onto the canvas.
2. Wire the I2C lines to **GPIO21/GPIO22**. Connect LED to **GPIO5** and Buzzer to **GPIO15**.
3. Paste the code and click **Run**.
4. Adjust the pitch/roll sliders on the MPU-6050 widget past 30 degrees to trigger the alarm.

## Expected Output
Serial Monitor:
```
Tilt Alarm Station ready.
Pitch: 5.2 | Roll: -2.0
Pitch: 32.4 | Roll: -2.1
!! TILT WARNING ALARM !!
```

## Expected Canvas Behavior
* While the virtual sensor is relatively level (under 30°), the LED and buzzer widgets are off.
* Adjusting the virtual sensor sliders past 30° on either axis turns ON the LED and buzzer widgets.
* Returning the sensor to a flat level position shuts off the alarm immediately.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `abs(pitch)` | Converts negative angle readings (tilting forward/left) to positive numbers for easier range checks. |
| `absPitch > TILT_LIMIT_DEG` | Evaluates if the device has tilted past the 30° limit on the pitch axis. |
| `while (1)` inside setup | Fail-safe: if the accelerometer is missing, halts operations and flashes the LED to warn the operator. |

## Hardware & Safety Concept: Industrial Tilt Warnings and Rollover Protection
In heavy machinery (e.g. cranes, excavators, forklifts), tilt monitoring is essential. If the machine travels onto an incline exceeding safety bounds, the control computer triggers alarms to notify the operator and can automatically lock controls to prevent a rollover hazard.

## Try This! (Challenges)
1. **Directional LED Indicators**: Wire 4 LEDs (Up, Down, Left, Right) to indicate which direction the sensor is tilting.
2. **Dynamic Alarm Pitch**: If using a passive buzzer, increase the tone pitch (Hz) as the tilt angle increases.
3. **Chime Mute Timer**: Allow the user to press a button on GPIO 4 to silence the buzzer for 15 seconds while tilting.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| System halts with flashing LED at boot | MPU-6050 initialisation failure | Check the I2C wiring (SDA -> GPIO 21, SCL -> GPIO 22) |
| Alarm triggers when resting flat | Sensor offsets are not calibrated | Subtract resting values or calculate baseline offsets during `setup()` |
| Alarm is laggy | Too much delay in loop | Ensure the loop runs at least at 10Hz (`delay(100)`) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [93 - ESP32 MPU-6050 Accelerometer Serial Logs](93-esp32-mpu6050-accelerometer-serial-logs.md)
- [94 - ESP32 MPU-6050 Gyroscope Position HUD](94-esp32-mpu6050-gyroscope-position-hud.md)
- [28 - ESP32 Tilt Sensor Level Indicator](../beginner/28-esp32-tilt-sensor-level-indicator.md)
