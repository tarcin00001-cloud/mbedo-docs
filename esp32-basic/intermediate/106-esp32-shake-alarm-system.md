# 106 - ESP32 Shake Alarm System

Build a motion-activated security alarm that uses an MPU-6050 accelerometer to detect physical shaking spikes, latches the alarm state, and requires a push button to reset.

## Goal
Learn how to capture rapid acceleration changes (shake gestures), implement a latching software state machine, and use a tactile reset button.

## What You Will Build
An MPU-6050 accelerometer is connected via I2C. A push button on GPIO 4 serves as the alarm reset, and a buzzer on GPIO 15 is the alarm output. If the acceleration force on the X or Y axis spikes above a threshold (due to shaking), the alarm latches ON. The buzzer continues to sound until the user presses the reset button.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MPU-6050 Accelerometer & Gyroscope | `mpu6050` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 Module | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| MPU-6050 Module | SDA | GPIO21 | Blue | I2C Data |
| MPU-6050 Module | SCL | GPIO22 | Yellow | I2C Clock |
| Push Button | Pin 1 | 3V3 | Red | Power |
| Push Button | Pin 2 | GPIO4 | Yellow | Reset Input |
| 10 kΩ Resistor | Leg 1 / Leg 2 | GPIO4 / GND | White / Black | Pull-down resistor |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Alarm output |
| Active Buzzer | GND (−) | GND | Black | Ground reference |

> **Wiring tip:** The reset button requires a 10 kΩ pull-down resistor to GND on GPIO 4. Connect the active buzzer directly to GPIO 15.

## Code
```cpp
// MPU-6050 Shake Alarm System
#include <Wire.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>

const int RESET_BTN = 4;
const int BUZZER_PIN = 15;

// Acceleration threshold to trigger shake alarm (in m/s^2)
// Excludes static gravity (~9.8 m/s^2)
const float SHAKE_THRESHOLD = 15.0; 

Adafruit_MPU6050 mpu;

bool alarmState = false;

void setup() {
  Serial.begin(115200);
  
  pinMode(RESET_BTN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  digitalWrite(BUZZER_PIN, LOW);
  
  if (!mpu.begin()) {
    Serial.println("MPU-6050 connection error!");
    while (1) {}
  }
  
  // Set accelerometer range
  mpu.setAccelerometerRange(MPU6050_RANGE_8_G);
  
  Serial.println("Shake Alarm Station ready.");
}

void loop() {
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);
  
  // Read raw acceleration forces
  float ax = a.acceleration.x;
  float ay = a.acceleration.y;
  float az = a.acceleration.z;
  
  // Check if shake occurs on X or Y axis
  bool shakeDetected = (abs(ax) > SHAKE_THRESHOLD || abs(ay) > SHAKE_THRESHOLD);
  
  if (shakeDetected && !alarmState) {
    Serial.println("!! ALARM TRIGGERED: SHAKE DETECTED !!");
    alarmState = true;
  }
  
  // Check if reset button is pressed
  if (digitalRead(RESET_BTN) == HIGH && alarmState) {
    Serial.println("Reset button pressed. Alarm cleared.");
    alarmState = false;
    digitalWrite(BUZZER_PIN, LOW);
  }
  
  // Drive buzzer based on latched state
  if (alarmState) {
    // Pulsing siren sound
    digitalWrite(BUZZER_PIN, HIGH);
    delay(100);
    digitalWrite(BUZZER_PIN, LOW);
    delay(100);
  } else {
    delay(50); // Small poll delay when idle
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MPU-6050**, **Button**, and **Buzzer** onto the canvas.
2. Wire MPU-6050 I2C to **GPIO21/GPIO22**, Button to **GPIO4**, and Buzzer to **GPIO15**.
3. Paste the code and click **Run**.
4. Slide the MPU-6050 X or Y acceleration past 15 on the canvas. Watch the buzzer begin to pulse.
5. Click the button to reset the buzzer alarm.

## Expected Output
Serial Monitor:
```
Shake Alarm Station ready.
!! ALARM TRIGGERED: SHAKE DETECTED !!
Reset button pressed. Alarm cleared.
```

## Expected Canvas Behavior
* While the MPU-6050 is static, the buzzer widget stays OFF.
* Sliding the X or Y acceleration sliders to values above 15.0 or below -15.0 immediately triggers the buzzer to pulse.
* The buzzer continues to beep even if the sliders are returned to 0.
* Clicking the button widget clears the alarm and silences the buzzer.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `abs(ax) > SHAKE_THRESHOLD` | Detects high acceleration spikes in either positive or negative directions. |
| `alarmState = true` | Latches the alarm state to true (latching memory). |
| `digitalRead(RESET_BTN) == HIGH` | Clears the latching alarm state when the button is pressed. |

## Hardware & Safety Concept: Latching Relays and Manual Resets
Security systems use latching logic to ensure that if an intrusion occurs (e.g. window glass breaking or fence shaking), the alarm remains active even after the intruder stops moving or runs away. This alerts safety monitors that a breach occurred. The system can only be turned off using a manual reset switch or secure code console.

## Try This! (Challenges)
1. **Z-Axis Drop Detector**: Include Z-axis measurements to trigger the alarm if the sensor experiences freefall (Z drops close to 0 \(m/s^2\)).
2. **LED Status Indicator**: Add a Red LED on GPIO 5 that flashes alongside the buzzer when the alarm is active.
3. **Double Tap Reset**: Require the user to double-click the reset button within 500 ms to clear the alarm.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers randomly | Sensitivity limit set too low | Increase the `SHAKE_THRESHOLD` variable in the code |
| Reset button doesn't clear alarm | Button pull-down resistor missing | Ensure a physical 10 kΩ resistor is pulling GPIO 4 down to GND |
| Buzzer click sounds weak | Active buzzer driven by analog/PWM | Verify the buzzer pin is configured for digital writes |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [93 - ESP32 MPU-6050 Accelerometer Serial Logs](93-esp32-mpu6050-accelerometer-serial-logs.md)
- [95 - ESP32 MPU-6050 Tilt Sensor Alarm](95-esp32-mpu6050-tilt-sensor-alarm.md)
- [29 - ESP32 Vibration Sensor Latch Alarm](../beginner/29-esp32-vibration-sensor-latch-alarm.md)
// 106-esp32-shake-alarm-system.md
