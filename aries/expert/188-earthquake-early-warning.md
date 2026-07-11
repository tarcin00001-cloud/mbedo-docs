# 188 - Earthquake Early Warning

Detect seismic vibration events using an MPU-6050 accelerometer/gyroscope module, classify tremor intensity, and trigger graduated multi-output alerts — LED colour change, buzzer alarm, and Serial log — when acceleration thresholds are exceeded.

## Goal

Learn how to poll an I2C accelerometer at high frequency, compute an acceleration magnitude from X/Y/Z components without arrays or loops, implement threshold-based alert state machines, and produce graduated responses proportional to measured seismic intensity.

## What You Will Build

The ARIES v3 board communicates with an MPU-6050 over I2C (SDA = GPIO 4, SCL = GPIO 5). Each `loop()` reads raw 16-bit accelerometer registers, converts them to g-force values, computes vector magnitude, and compares it against three alert thresholds (advisory, warning, alarm). Corresponding LED colours (Green / Yellow via Red+Green / Red) and buzzer cadence change with severity. All readings and alert states stream to the Serial Monitor.

## Parts Needed

| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MPU-6050 Accelerometer/Gyro | `mpu6050` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Red LED (onboard) | — | Yes | Yes |
| Green LED (onboard) | — | Yes | Yes |
| Warning LED | `led` | Yes | Yes |
| Jumper Wires | — | No | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 | VCC | 3V3 | Red | 3.3 V power |
| MPU-6050 | GND | GND | Black | Ground |
| MPU-6050 | SDA | GPIO 4 | Yellow | I2C data |
| MPU-6050 | SCL | GPIO 5 | Green | I2C clock |
| MPU-6050 | AD0 | GND | Black | I2C address = 0x68 |
| Active Buzzer | + | GPIO 14 | Orange | Alarm buzzer |
| Active Buzzer | − | GND | Black | Ground |
| Warning LED | Anode | GPIO 15 | Orange | External warning LED |
| Warning LED | Cathode | GND (via 220 Ω) | Black | Current-limiting resistor |

> **Wiring tip:** Pull SDA and SCL lines to 3.3 V through 4.7 kΩ resistors if your MPU-6050 breakout board does not already include them (most GY-521 modules do). Connect AD0 to GND to set I2C address 0x68. If AD0 is connected to 3.3 V the address becomes 0x69 — update `MPU_ADDR` accordingly.

## Code

```cpp
// 188 - Earthquake Early Warning
// MPU-6050 I2C accelerometer threshold detection + graduated alert outputs
#include <Wire.h>

#define MPU_ADDR   0x68
#define PIN_LED_R  23    // Onboard Red LED
#define PIN_LED_G  24    // Onboard Green LED
#define PIN_LED_W  15    // External warning LED
#define PIN_BUZZ   14    // Active buzzer

// Alert thresholds in g-force magnitude (1g = normal gravity)
// Magnitude = sqrt(ax^2 + ay^2 + az^2); at rest ~1.0g
#define THRESH_ADVISORY  1.15   // light tremor
#define THRESH_WARNING   1.40   // moderate tremor
#define THRESH_ALARM     1.80   // strong tremor

float ax = 0.0, ay = 0.0, az = 0.0;
float magnitude   = 0.0;
int   alertLevel  = 0;   // 0=normal 1=advisory 2=warning 3=alarm
int   buzzPhase   = 0;
int   buzzTick    = 0;
int   logTick     = 0;

void setup() {
  Serial.begin(115200);
  Wire.begin();

  // Wake MPU-6050 from sleep
  Wire.beginTransmission(MPU_ADDR);
  Wire.write(0x6B);   // PWR_MGMT_1 register
  Wire.write(0x00);   // Clear sleep bit
  Wire.endTransmission(true);

  pinMode(PIN_LED_R, OUTPUT);
  pinMode(PIN_LED_G, OUTPUT);
  pinMode(PIN_LED_W, OUTPUT);
  pinMode(PIN_BUZZ,  OUTPUT);

  digitalWrite(PIN_LED_R, LOW);
  digitalWrite(PIN_LED_G, HIGH);   // Green = all clear
  digitalWrite(PIN_LED_W, LOW);
  digitalWrite(PIN_BUZZ,  LOW);

  Serial.println("Earthquake Early Warning System Online.");
  Serial.println("Thresholds: Advisory=1.15g  Warning=1.40g  Alarm=1.80g");
}

void loop() {
  // --- Read raw accelerometer registers ---
  Wire.beginTransmission(MPU_ADDR);
  Wire.write(0x3B);   // ACCEL_XOUT_H register
  Wire.endTransmission(false);
  Wire.requestFrom(MPU_ADDR, 6, true);

  int rawX = (Wire.read() << 8) | Wire.read();
  int rawY = (Wire.read() << 8) | Wire.read();
  int rawZ = (Wire.read() << 8) | Wire.read();

  // Convert to g-force (±2g range default: 16384 LSB/g)
  ax = rawX / 16384.0;
  ay = rawY / 16384.0;
  az = rawZ / 16384.0;

  // Magnitude (avoid sqrt — use squared comparison where possible,
  // but sqrt is acceptable in interpreted mode for display)
  magnitude = sqrt(ax * ax + ay * ay + az * az);

  // --- Determine alert level ---
  if (magnitude >= THRESH_ALARM) {
    alertLevel = 3;
  } else if (magnitude >= THRESH_WARNING) {
    alertLevel = 2;
  } else if (magnitude >= THRESH_ADVISORY) {
    alertLevel = 1;
  } else {
    alertLevel = 0;
  }

  // --- LED state machine ---
  if (alertLevel == 0) {
    digitalWrite(PIN_LED_G, HIGH);
    digitalWrite(PIN_LED_R, LOW);
    digitalWrite(PIN_LED_W, LOW);
  }
  if (alertLevel == 1) {
    digitalWrite(PIN_LED_G, HIGH);
    digitalWrite(PIN_LED_R, HIGH);
    digitalWrite(PIN_LED_W, LOW);
  }
  if (alertLevel == 2) {
    digitalWrite(PIN_LED_G, LOW);
    digitalWrite(PIN_LED_R, HIGH);
    digitalWrite(PIN_LED_W, HIGH);
  }
  if (alertLevel == 3) {
    // Rapid flash for alarm level
    int flashState = (buzzTick / 2) % 2;
    digitalWrite(PIN_LED_R, flashState);
    digitalWrite(PIN_LED_W, flashState);
    digitalWrite(PIN_LED_G, LOW);
  }

  // --- Buzzer cadence state machine ---
  buzzTick++;
  if (alertLevel == 0) {
    digitalWrite(PIN_BUZZ, LOW);
  }
  if (alertLevel == 1) {
    // Slow beep: on for 5 ticks, off for 45 ticks (1-second period)
    digitalWrite(PIN_BUZZ, ((buzzTick % 50) < 5) ? HIGH : LOW);
  }
  if (alertLevel == 2) {
    // Medium beep: on for 10 ticks, off for 15 ticks
    digitalWrite(PIN_BUZZ, ((buzzTick % 25) < 10) ? HIGH : LOW);
  }
  if (alertLevel == 3) {
    // Continuous alarm
    digitalWrite(PIN_BUZZ, HIGH);
  }

  // --- Serial log every 50 ticks (~500 ms) ---
  logTick++;
  if (logTick >= 50) {
    logTick = 0;
    Serial.print("aX:");
    Serial.print(ax, 3);
    Serial.print("g  aY:");
    Serial.print(ay, 3);
    Serial.print("g  aZ:");
    Serial.print(az, 3);
    Serial.print("g  |Mag|:");
    Serial.print(magnitude, 3);
    Serial.print("g  Alert:");
    if (alertLevel == 0) Serial.println("NORMAL");
    if (alertLevel == 1) Serial.println("ADVISORY");
    if (alertLevel == 2) Serial.println("WARNING");
    if (alertLevel == 3) Serial.println("** ALARM **");
  }

  delay(10);
}
```

## What to Click in MbedO

1. Drag **VEGA ARIES v3**, **MPU-6050**, **Active Buzzer**, and a **Warning LED** onto the canvas.
2. Wire MPU-6050: **SDA → GPIO 4**, **SCL → GPIO 5**, **VCC → 3V3**, **GND → GND**, **AD0 → GND**.
3. Wire Buzzer **+ → GPIO 14**, **− → GND**.
4. Wire Warning LED **Anode → GPIO 15**, **Cathode → GND** (via 220 Ω).
5. Paste the code and select **Interpreted Mode**.
6. Click **Run**.
7. In the MPU-6050 widget, move the **X-axis** or **Y-axis** acceleration sliders above 1.15 to trigger Advisory.
8. Move sliders above 1.40 for Warning; above 1.80 for Alarm.
9. Observe LED colour changes and buzzer cadence matching each level in the Serial Monitor.

## Expected Output

Serial Monitor:
```
Earthquake Early Warning System Online.
Thresholds: Advisory=1.15g  Warning=1.40g  Alarm=1.80g
aX:0.012g  aY:-0.008g  aZ:0.998g  |Mag|:0.999g  Alert:NORMAL
aX:0.521g  aY:0.412g  aZ:0.998g  |Mag|:1.230g  Alert:ADVISORY
aX:1.021g  aY:0.812g  aZ:0.998g  |Mag|:1.620g  Alert:WARNING
aX:1.521g  aY:1.312g  aZ:0.998g  |Mag|:2.148g  Alert:** ALARM **
```

## Expected Canvas Behavior

* At rest, the Green LED is ON; buzzer silent; Serial shows NORMAL.
* Advisory level: Green + Red LEDs both ON (yellow appearance); buzzer beeps slowly every second.
* Warning level: Red LED + Warning LED ON; buzzer beeps rapidly.
* Alarm level: Red LED + Warning LED flash rapidly; buzzer sounds continuously.

## Code Walkthrough

| Line | Check / Action |
| --- | --- |
| `Wire.write(0x6B); Wire.write(0x00)` | Clears the sleep bit in MPU-6050 PWR_MGMT_1 register to wake the chip. |
| `Wire.write(0x3B)` | Points the I2C read pointer to ACCEL_XOUT_H, the first accelerometer register. |
| `Wire.requestFrom(MPU_ADDR, 6, true)` | Reads 6 consecutive bytes: two each for X, Y, Z high/low register pairs. |
| `rawX = (Wire.read() << 8) \| Wire.read()` | Combines high and low bytes into a signed 16-bit integer. |
| `ax = rawX / 16384.0` | Converts to g-force: ±2g range uses 16384 LSB per g. |
| `magnitude = sqrt(ax*ax + ay*ay + az*az)` | Euclidean vector magnitude; at rest ≈ 1.0 g (Earth gravity on Z). |
| `alertLevel = 3 / 2 / 1 / 0` | Cascade comparison sets the highest matching threshold level. |
| `buzzTick % 50 < 5` | Modulo division creates a repeating beep pattern without a loop. |

## Hardware & Safety Concept

**Inertial Measurement:** The MPU-6050 contains a MEMS (Micro-Electro-Mechanical System) accelerometer. Tiny silicon proof masses deflect under acceleration, changing the capacitance of comb-finger structures. The onboard ADC converts the capacitance change to a 16-bit integer. At rest, the sensor measures Earth's gravitational acceleration (~1 g) on the Z-axis. Seismic events add lateral (X, Y) acceleration components, increasing the total vector magnitude above 1 g.

**Graduated Response:** Real earthquake early warning systems (e.g., Japan's BOUSAI) emit graduated alerts as P-waves arrive before destructive S-waves. This project simulates that hierarchy: Advisory prepares users, Warning triggers protective actions, Alarm demands immediate evacuation. The state machine ensures outputs match the current threat level without any latency-inducing loops.

## Try This! (Challenges)

1. **Peak magnitude logger**: Track and print the maximum magnitude recorded since power-on using a `float peakMag` state variable updated with `if (magnitude > peakMag) peakMag = magnitude;`.
2. **Gyroscope integration**: Add reading of the gyroscope registers (0x43–0x48) and print rotational rates alongside linear acceleration to detect rotational tremors.

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| All readings show 0.000 g | MPU-6050 still in sleep / I2C wiring error | Verify SDA=GPIO4, SCL=GPIO5; confirm AD0 to GND for address 0x68. |
| Magnitude stuck at 1.0 g | Sensor working but stationary | Move the X/Y sliders in the MbedO MPU-6050 widget to simulate vibration. |
| Buzzer never stops | alertLevel never drops below 3 | Reduce the axis slider values to bring magnitude below THRESH_ALARM. |
| Alarm triggers at rest | Threshold too low / sensor offset | Increase THRESH_ADVISORY; calibrate by printing magnitude for 10 s at rest and setting threshold 0.1 g above the observed resting mean. |

## Mode Notes

This project runs in MbedO **interpreted C++ mode**.

## Related Projects

- [187 - Smart Irrigation System](187-smart-irrigation-system.md)
- [189 - Precision Animal Feeder](189-precision-animal-feeder.md)
- [200 - Smart City Node (Capstone)](200-risc-v-superproject-smart-city-node.md)
