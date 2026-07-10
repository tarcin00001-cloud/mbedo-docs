# 107 - Shake Detector

Read X and Y axis acceleration from an MPU-6050 IMU and trigger a combined buzzer-and-LED alert whenever a shake event is detected — a foundational pattern for step counters, drop detectors, and gesture sensors.

## Goal
Learn how to combine two axis readings with an OR condition to detect rapid movement in any horizontal direction, and how to coordinate multiple output devices in a single alert response.

## What You Will Build
Every 100 ms the sketch reads Ac_x and Ac_y. If either axis exceeds the shake threshold (indicating rapid acceleration in that direction), the LED flashes and the buzzer sounds a short beep, then both turn off after 300 ms. The Serial Monitor prints each axis value and flags shake events.

**Why these pins?** The MPU-6050 uses I2C on A4/A5. The LED goes to `D7` with a 220 ohm resistor. The buzzer goes to `D8` for `tone()` generation.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| MPU-6050 IMU | `mpu6050` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 ohm resistor | `resistor` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes |

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
| Passive Buzzer | + | D8 | PWM tone output |
| Passive Buzzer | − | GND | Ground reference |

## Code
```cpp
#include <Wire.h>

const int MPU_ADDR       = 0x68;
const int LED_PIN        = 7;
const int BUZZER_PIN     = 8;

// Shake threshold in raw units — ~2g lateral acceleration
const int SHAKE_THRESHOLD = 8000;

int16_t Ac_x, Ac_y, Ac_z;

void setup() {
  Serial.begin(9600);
  Wire.begin();

  // Wake up MPU-6050
  Wire.beginTransmission(MPU_ADDR);
  Wire.write(0x6B);
  Wire.write(0);
  Wire.endTransmission(true);

  pinMode(LED_PIN,    OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(LED_PIN,    LOW);
  digitalWrite(BUZZER_PIN, LOW);

  Serial.println("Shake Detector Ready");
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

  bool shakeX = (Ac_x > SHAKE_THRESHOLD || Ac_x < -SHAKE_THRESHOLD);
  bool shakeY = (Ac_y > SHAKE_THRESHOLD || Ac_y < -SHAKE_THRESHOLD);

  if (shakeX || shakeY) {
    Serial.println(" | SHAKE DETECTED!");

    digitalWrite(LED_PIN, HIGH);
    tone(BUZZER_PIN, 1500, 200);
    delay(300);
    digitalWrite(LED_PIN, LOW);
    noTone(BUZZER_PIN);
  } else {
    Serial.println(" | Stable");
    digitalWrite(LED_PIN,    LOW);
    noTone(BUZZER_PIN);
  }

  delay(100);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **MPU-6050 IMU**, **LED** with **220 ohm resistor**, and **Passive Buzzer** onto the canvas.
2. Connect MPU-6050: **VCC** to **3.3V**, **GND** to **GND**, **SDA** to **A4**, **SCL** to **A5**, **AD0** to **GND**.
3. Connect LED anode through 220 ohm resistor to **D7**; cathode to **GND**.
4. Connect Buzzer: **+** to **D8**, **−** to **GND**.
5. Paste the code into the editor.
6. Use the default Arduino interpreted runtime.
7. Click **Run**.
8. Double-click the MPU-6050 on the canvas and rapidly drag the **Ac_x** or **Ac_y** slider beyond ±8000 — the LED should flash and the buzzer should beep. Release to let it return to stable.

## Expected Output

Terminal:
```
Shake Detector Ready
Ac_x: 320 | Ac_y: -80 | Stable
Ac_x: 9500 | Ac_y: 200 | SHAKE DETECTED!
Ac_x: -8200 | Ac_y: 400 | SHAKE DETECTED!
Ac_x: 150 | Ac_y: 60 | Stable
```

## Expected Canvas Behavior
| Ac_x | Ac_y | shakeX | shakeY | LED | Buzzer |
| --- | --- | --- | --- | --- | --- |
| 300 | −80 | false | false | OFF | Silent |
| 9500 | 200 | true | false | ON (flash) | 200 ms beep |
| 200 | −8500 | false | true | ON (flash) | 200 ms beep |
| 9200 | 8800 | true | true | ON (flash) | 200 ms beep |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Ac_x > SHAKE_THRESHOLD \|\| Ac_x < -SHAKE_THRESHOLD` | Checks for shake in both the positive and negative X directions. |
| `shakeX \|\| shakeY` | Logical OR — triggers the alert if either horizontal axis exceeds the threshold. |
| `tone(BUZZER_PIN, 1500, 200)` | Plays a 1500 Hz beep for 200 ms; the duration parameter stops it automatically. |
| `delay(300)` inside the shake block | Keeps the LED on for 300 ms to make the flash visible before turning it off. |
| `noTone(BUZZER_PIN)` in the else branch | Ensures any lingering tone is silenced on every stable reading. |

## Hardware & Safety Concept: Dynamic vs Static Acceleration
Accelerometers measure the **total** force (in units of g) along each axis — this includes both the static gravitational component and dynamic accelerations from movement. At rest, only gravity appears.
- A shake event adds a dynamic acceleration spike on top of the gravity baseline, pushing the total reading well above 1g (16384 raw units).
- The threshold of 8000 represents approximately 0.5g lateral acceleration above the resting baseline — sensitive enough to detect a sharp tap but resistant to slow tilts.
- Very short shakes (under 100 ms) may be missed at a 100 ms loop rate; reducing `delay(100)` increases responsiveness but also increases serial output volume.

## Try This! (Challenges)
1. **Shake Counter**: Add an integer `shakeCount` variable that increments on each detected shake event and print `"Shake #N"` to serial so you can count events over time.
2. **Axis Identification**: In the serial output, print `"SHAKE X"` or `"SHAKE Y"` or `"SHAKE XY"` depending on which combination triggered, helping you understand the physical movement direction.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Constant shake alerts at rest | Threshold too low | Increase `SHAKE_THRESHOLD` to 10000–12000 to reduce false positives. |
| No shake detected even at high slider values | MPU-6050 still sleeping | Confirm the `Wire.write(0x6B); Wire.write(0)` block runs in `setup()`. |
| LED stays ON permanently | `LOW` not set after 300 ms | Ensure `digitalWrite(LED_PIN, LOW)` runs after the `delay(300)` inside the `if` block. |
| Buzzer keeps beeping | `noTone()` missing | Verify `noTone(BUZZER_PIN)` is called in the `else` branch. |

## Mode Notes
All patterns used here (`Wire.begin`, `Wire.requestFrom`, `Wire.read`, `tone()`, `noTone()`, `digitalWrite`, `if/else`) are fully supported by MbedO interpreted mode.

## Related Projects
- [105 - Tilt Alarm](105-tilt-alarm.md)
- [106 - Tilt LED](106-tilt-led.md)
- [91 - IMU Serial](91-imu-serial.md)
- [103 - Smart Doorbell](103-smart-doorbell.md)
