# 95 - Pico MPU6050 Tilt

Build a solid-state tilt alarm system that triggers alerts when tilted beyond a threshold.

## Goal
Learn how to implement physical tilt-angle detection using raw accelerometer coordinate vectors and trigger safety outputs.

## What You Will Build
A structural tilt monitoring alarm:
- **MPU-6050 IMU (GP4 SDA, GP5 SCL)**: Monitors X and Y axis tilt acceleration.
- **Active Buzzer (GP14)**: Sounds a rapid alert beep if the tilt exceeds the threshold.
- **Red LED (GP15)**: Illuminates during the tilt alert phase.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MPU-6050 IMU Module | `mpu6050` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MPU-6050 | VCC | 3.3V | Power supply |
| MPU-6050 | SDA | GP4 | Shared I2C Data |
| MPU-6050 | SCL | GP5 | Shared I2C Clock |
| MPU-6050 | GND | GND | Ground return |
| Red LED | Anode | GP15 | Visual indicator |
| Red LED | Cathode | GND | Ground return via resistor |
| Active Buzzer | VCC (+) | GP14 | Acoustic indicator |
| Active Buzzer | GND (-) | GND | Ground return |

## Code
```cpp
#include <Wire.h>

// Raw shims required by interpreted mode
int16_t Ac_x, Ac_y, Ac_z;
int16_t Gy_x, Gy_y, Gy_z;

const int MPU_addr = 0x68;
const int LED_PIN  = 15;
const int BUZZ_PIN = 14;

// Tilt limits: raw gravity units (16384 = 1g)
// 8000 corresponds to approx. 30 degrees of tilt
const int TILT_LIMIT = 8000;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  // Wake up MPU-6050
  Wire.beginTransmission(MPU_addr);
  Wire.write(0x6B);
  Wire.write(0);
  Wire.endTransmission(true);

  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZ_PIN, OUTPUT);
  
  digitalWrite(LED_PIN, LOW);
  digitalWrite(BUZZ_PIN, LOW);
}

void loop() {
  Wire.beginTransmission(MPU_addr);
  Wire.write(0x3B);
  Wire.endTransmission(false);
  Wire.requestFrom(MPU_addr, 14, true);

  // Read raw acceleration values
  Ac_x = Wire.read() << 8 | Wire.read();
  Ac_y = Wire.read() << 8 | Wire.read();
  Ac_z = Wire.read() << 8 | Wire.read();
  
  // Skip temp bytes
  int16_t tmp = Wire.read() << 8 | Wire.read();

  // Skip gyro bytes
  Gy_x = Wire.read() << 8 | Wire.read();
  Gy_y = Wire.read() << 8 | Wire.read();
  Gy_z = Wire.read() << 8 | Wire.read();

  // Calculate absolute acceleration deviation on X and Y axes
  int absX = Ac_x;
  if (absX < 0) { absX = -absX; }
  
  int absY = Ac_y;
  if (absY < 0) { absY = -absY; }

  // Trigger alert if tilt limit is breached
  if (absX > TILT_LIMIT || absY > TILT_LIMIT) {
    digitalWrite(LED_PIN, HIGH);
    digitalWrite(BUZZ_PIN, HIGH);
    delay(100);
    digitalWrite(BUZZ_PIN, LOW);
    delay(100);
  } else {
    digitalWrite(LED_PIN, LOW);
    digitalWrite(BUZZ_PIN, LOW);
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MPU-6050 Sensor**, **Red LED**, and **Active Buzzer** onto the canvas.
2. Connect MPU-6050 to GP4 (SDA) and GP5 (SCL). Connect LED to GP15, Buzzer to GP14.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the rotation of the MPU-6050 on canvas past 30 degrees and monitor the alarm indicators.

## Expected Output

Terminal:
```
Simulation active. Solid-state tilt alarm monitor active.
```

## Expected Canvas Behavior
| MPU-6050 Orientation | LED (GP15) | Buzzer (GP14) | Status |
| --- | --- | --- | --- |
| Flat (Normal) | LOW | LOW | Safe |
| Tilted X > 30° | HIGH | Pulsing | **TILT ALARM ACTIVE** |
| Tilted Y > 30° | HIGH | Pulsing | **TILT ALARM ACTIVE** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `absX = -absX` | Absolute value conversion helper logic, ensuring tilt checks work in both positive and negative directions (e.g. left and right). |

## Hardware & Safety Concept: Structural Tilt Monitoring
In civil engineering (bridges, towers, retaining walls), structural monitoring systems use highly sensitive inclinometers (MEMS tilt sensors) to monitor shifts. If a tower or retaining wall tilts by even a fraction of a degree, it represents a structural hazard. Early warning systems detect these trends, log them over I2C networks, and sound alarms to evacuate the site.

## Try This! (Challenges)
1. **Latching Alarm**: Modify the code so that once a tilt alarm is triggered, it stays ON forever until the Pico is power-cycled.
2. **Dynamic Alarm rate**: Make the buzzer beep faster as the tilt angle increases.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers when flat | Sensor calibration offset | Cheap IMU sensors have offset errors. Add a constant calibration offset value to the raw values inside setup to nullify sensor drift. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode. Variables representing MPU-6050 inputs must be named exactly `Ac_x`, `Ac_y`, `Ac_z`, `Gy_x`, `Gy_y`, `Gy_z`.

## Related Projects
- [28 - Pico Tilt Switch](../../beginner/28-pico-tilt-switch.md)
- [93 - Pico MPU6050 Serial](93-pico-mpu6050-serial.md)
