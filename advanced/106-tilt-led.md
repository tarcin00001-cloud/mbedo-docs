# 106 - Tilt LED

Read the Z-axis acceleration from an MPU-6050 IMU sensor and light an LED whenever the board is tilted significantly from vertical — a simple orientation indicator or level-crossing light.

## Goal
Learn how to use a single IMU axis value with a threshold `if` statement to drive a digital output, understanding how raw accelerometer counts relate to physical orientation.

## What You Will Build
Every 300 ms the sketch reads `Ac_z` (Z-axis acceleration). When the board lies flat, Ac_z is close to 16384 (1g). When the board is tilted so Ac_z drops below a threshold (meaning gravity is no longer pointing straight down through the Z axis), the LED lights up. When returned to flat the LED turns off. The Serial Monitor continuously prints all three axis values.

**Why these pins?** The MPU-6050 communicates over I2C — SDA to `A4` and SCL to `A5`. The LED goes to `D7` through a 220 ohm current-limiting resistor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| MPU-6050 IMU | `mpu6050` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 ohm resistor | `resistor` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| MPU-6050 | VCC | 3.3V | Power supply (3.3 V only!) |
| MPU-6050 | GND | GND | Ground reference |
| MPU-6050 | SDA | A4 | I2C Data |
| MPU-6050 | SCL | A5 | I2C Clock |
| MPU-6050 | AD0 | GND | Sets I2C address to 0x68 |
| LED | Anode (+) | D7 | Through 220 ohm resistor |
| LED | Cathode (−) | GND | Ground reference |

## Code
```cpp
#include <Wire.h>

const int MPU_ADDR = 0x68;
const int LED_PIN  = 7;

// When Ac_z drops below this the board is noticeably tilted
const int FLAT_THRESHOLD = 12000;

int16_t Ac_x, Ac_y, Ac_z;

void setup() {
  Serial.begin(9600);
  Wire.begin();

  // Wake up the MPU-6050
  Wire.beginTransmission(MPU_ADDR);
  Wire.write(0x6B);
  Wire.write(0);
  Wire.endTransmission(true);

  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);

  Serial.println("Tilt LED Ready");
}

void loop() {
  Wire.beginTransmission(MPU_ADDR);
  Wire.write(0x3B);
  Wire.endTransmission(false);
  Wire.requestFrom(MPU_ADDR, 6, true);

  Ac_x = (Wire.read() << 8) | Wire.read();
  Ac_y = (Wire.read() << 8) | Wire.read();
  Ac_z = (Wire.read() << 8) | Wire.read();

  Serial.print("Ac_x: ");
  Serial.print(Ac_x);
  Serial.print(" | Ac_y: ");
  Serial.print(Ac_y);
  Serial.print(" | Ac_z: ");
  Serial.println(Ac_z);

  if (Ac_z < FLAT_THRESHOLD) {
    digitalWrite(LED_PIN, HIGH);
    Serial.println("Tilted - LED ON");
  } else {
    digitalWrite(LED_PIN, LOW);
    Serial.println("Flat   - LED OFF");
  }

  delay(300);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **MPU-6050 IMU**, **LED**, and **220 ohm resistor** onto the canvas.
2. Connect MPU-6050: **VCC** to **3.3V**, **GND** to **GND**, **SDA** to **A4**, **SCL** to **A5**, **AD0** to **GND**.
3. Connect LED anode through the 220 ohm resistor to **D7**; LED cathode to **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Double-click the MPU-6050 on the canvas and drag the **Ac_z** slider below 12000 — the LED should light up. Return it above 12000 and the LED should turn off.

## Expected Output

Terminal:
```
Tilt LED Ready
Ac_x: 256 | Ac_y: -128 | Ac_z: 16100
Flat   - LED OFF
Ac_x: 4200 | Ac_y: 800 | Ac_z: 9500
Tilted - LED ON
```

## Expected Canvas Behavior
| Ac_z Value | Orientation | LED |
| --- | --- | --- |
| 12000 to 16384 | Flat / near-flat | OFF |
| Below 12000 | Tilted | ON |
| Negative | Upside down | ON |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Wire.write(0x3B)` | Selects the ACCEL_XOUT_H register; the MPU auto-increments through all six data bytes. |
| `(Wire.read() << 8) \| Wire.read()` | Assembles a signed 16-bit value from two consecutive bytes (big-endian format). |
| `Ac_z < FLAT_THRESHOLD` | True when the downward gravity component on Z falls below 12000 — indicating tilt. |
| `digitalWrite(LED_PIN, HIGH)` | Turns the LED on when the tilt condition is met. |

## Hardware & Safety Concept: Gravity Vector and Accelerometer Axes
At rest the MPU-6050 measures only the gravitational acceleration (1g ≈ 9.81 m/s²). For a flat board, almost all of this 1g appears on the Z axis. As the board tilts, gravity is distributed across all three axes. The relationship is governed by trigonometry: Ac_z = 16384 × cos(θ), where θ is the tilt angle from horizontal.
- At 45° tilt: Ac_z ≈ 11585 (cos 45° × 16384).
- At 90° tilt: Ac_z ≈ 0 (all gravity on X or Y axis).
- The FLAT_THRESHOLD of 12000 corresponds to roughly a 43° tilt angle.

## Try This! (Challenges)
1. **Angle Display**: Use the Serial Monitor to print the approximate tilt angle calculated as `90 - (int)((Ac_z / 16384.0) * 90)` degrees, giving a more intuitive reading than raw ADC counts.
2. **Two LEDs**: Add a second LED to `D6` and light it only when `Ac_z` is negative (board is upside down), creating a three-state orientation indicator: flat (off/off), tilted (on/off), inverted (off/on).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED is always ON | Threshold too high | Lower `FLAT_THRESHOLD` to 8000 and check if the board is actually flat. |
| LED is always OFF | MPU-6050 reading 0 | Verify the MPU-6050 wake-up code in `setup()` runs correctly. |
| All Ac values are 0 | I2C not initialised | Confirm `Wire.begin()` is called before any `Wire.beginTransmission()`. |
| LED flickers | Ac_z near threshold | Add a small hysteresis band (e.g., ON below 11500, OFF above 12500) using a state variable. |

## Mode Notes
All patterns used here (`Wire.begin`, `Wire.requestFrom`, `Wire.read`, `digitalWrite`, `if/else`) are fully supported by MbedO interpreted mode.

## Related Projects
- [105 - Tilt Alarm](105-tilt-alarm.md)
- [107 - Shake Detector](107-shake-detector.md)
- [91 - IMU Serial](91-imu-serial.md)
- [99 - Auto Night Light](99-auto-night-light.md)
