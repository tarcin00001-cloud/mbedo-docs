# 105 - Tilt Alarm

Read the X-axis acceleration from an MPU-6050 IMU sensor and sound a buzzer alarm whenever the board tilts beyond a set threshold — simulating a tamper detector or anti-theft tilt switch.

## Goal
Learn how to read raw accelerometer data from the MPU-6050 over I2C using the `Ac_x` variable alias, compare it against a threshold with an `if` statement, and drive a buzzer alarm.

## What You Will Build
Every 300 ms the sketch reads the X-axis acceleration value from the MPU-6050. When the absolute tilt value exceeds the threshold the buzzer sounds a continuous alarm tone and the Serial Monitor prints "TILT ALARM!". When tilt returns within bounds the buzzer stops and a normal reading is printed.

**Why these pins?** The MPU-6050 communicates over I2C — SDA to `A4` and SCL to `A5`. The buzzer goes to `D8`. No library is needed beyond `Wire.h`.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| MPU-6050 IMU | `mpu6050` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| MPU-6050 | VCC | 3.3V | Power supply (3.3 V only!) |
| MPU-6050 | GND | GND | Ground reference |
| MPU-6050 | SDA | A4 | I2C Data |
| MPU-6050 | SCL | A5 | I2C Clock |
| MPU-6050 | AD0 | GND | Sets I2C address to 0x68 |
| Passive Buzzer | + | D8 | PWM tone output |
| Passive Buzzer | − | GND | Ground reference |

## Code
```cpp
#include <Wire.h>

const int MPU_ADDR   = 0x68;
const int BUZZER_PIN = 8;

// Threshold in raw accelerometer units (~16384 = 1g at default ±2g range)
const int TILT_THRESHOLD = 6000;

int16_t Ac_x, Ac_y, Ac_z;

void setup() {
  Serial.begin(9600);
  Wire.begin();

  // Wake up the MPU-6050
  Wire.beginTransmission(MPU_ADDR);
  Wire.write(0x6B); // PWR_MGMT_1 register
  Wire.write(0);    // Set to 0 to wake from sleep
  Wire.endTransmission(true);

  pinMode(BUZZER_PIN, OUTPUT);

  Serial.println("Tilt Alarm Ready");
}

void loop() {
  // Request accelerometer data
  Wire.beginTransmission(MPU_ADDR);
  Wire.write(0x3B); // ACCEL_XOUT_H register
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

  if (Ac_x > TILT_THRESHOLD || Ac_x < -TILT_THRESHOLD) {
    tone(BUZZER_PIN, 1200);
    Serial.println("*** TILT ALARM! ***");
  } else {
    noTone(BUZZER_PIN);
    Serial.println("Normal orientation");
  }

  delay(300);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **MPU-6050 IMU**, and **Passive Buzzer** onto the canvas.
2. Connect MPU-6050: **VCC** to **3.3V**, **GND** to **GND**, **SDA** to **A4**, **SCL** to **A5**, **AD0** to **GND**.
3. Connect Buzzer: **+** to **D8**, **−** to **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Double-click the MPU-6050 on the canvas and drag the **Ac_x** slider beyond ±6000 — the buzzer should activate and the Serial Monitor should print "TILT ALARM!".

## Expected Output

Terminal:
```
Tilt Alarm Ready
Ac_x: 320 | Ac_y: -112 | Ac_z: 16200
Normal orientation
Ac_x: 8500 | Ac_y: 340 | Ac_z: 9100
*** TILT ALARM! ***
```

## Expected Canvas Behavior
| Ac_x Value | Condition | Buzzer | Serial Output |
| --- | --- | --- | --- |
| −5000 to +5999 | Within threshold | Silent | `Normal orientation` |
| > 6000 | Tilted right | 1200 Hz alarm | `*** TILT ALARM! ***` |
| < −6000 | Tilted left | 1200 Hz alarm | `*** TILT ALARM! ***` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Wire.write(0x6B); Wire.write(0)` | Writes 0 to the PWR_MGMT_1 register, waking the MPU-6050 from its default sleep state. |
| `Wire.write(0x3B)` | Points the internal register pointer to ACCEL_XOUT_H, the first of six accelerometer data bytes. |
| `(Wire.read() << 8) \| Wire.read()` | Combines two 8-bit bytes into one signed 16-bit integer for each axis value. |
| `Ac_x > TILT_THRESHOLD \|\| Ac_x < -TILT_THRESHOLD` | Checks both tilt directions — positive X (right) and negative X (left). |
| `tone(BUZZER_PIN, 1200)` | Starts a continuous 1200 Hz alarm tone; duration is omitted so it plays until `noTone()`. |

## Hardware & Safety Concept: MEMS Accelerometers
The MPU-6050 contains a Micro-Electro-Mechanical System (MEMS) accelerometer — a tiny silicon mass suspended by micro-springs. When the board tilts, gravity pulls the mass off-centre; capacitance changes between the mass and fixed plates are converted to a voltage, then to a digital value over I2C.
- At rest (flat), Ac_z reads approximately 16384 (1g pointing downward) and Ac_x / Ac_y read near 0.
- The default full-scale range is ±2g, so 16384 raw units = 1g.
- The MPU-6050 must be powered from 3.3 V — connecting VCC to 5 V will damage the chip on real hardware.

## Try This! (Challenges)
1. **Adjustable Threshold**: Wire a potentiometer to `A0` and use `map(analogRead(A0), 0, 1023, 1000, 16000)` to set `TILT_THRESHOLD` at runtime, allowing you to tune sensitivity without re-uploading.
2. **Axis Selector**: Monitor all three axes and print which axis triggered the alarm — for example, `"X-axis tilt!"` vs `"Y-axis tilt!"`.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| All values read 0 | MPU-6050 still sleeping | Confirm `Wire.write(0)` is sent to register `0x6B` in `setup()`. |
| Buzzer never stops | Threshold too low | Increase `TILT_THRESHOLD` (try 8000–12000) or verify the slider is within range. |
| Values wildly noisy | I2C pull-ups missing | Add 4.7 kΩ pull-up resistors from SDA and SCL to 3.3 V on real hardware. |
| Wrong I2C address | AD0 not grounded | Ground AD0 for address `0x68`; tie AD0 high for `0x69`. |

## Mode Notes
All patterns used here (`Wire.begin`, `Wire.beginTransmission`, `Wire.requestFrom`, `Wire.read`, `tone()`, `noTone()`, `if/else`) are fully supported by MbedO interpreted mode.

## Related Projects
- [104 - Water Level Station](104-water-level-station.md)
- [106 - Tilt LED](106-tilt-led.md)
- [107 - Shake Detector](107-shake-detector.md)
- [91 - IMU Serial](91-imu-serial.md)
