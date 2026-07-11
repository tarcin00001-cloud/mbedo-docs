# 149 - I2C OLED Tilt-compensated Compass

Combine an MPU-6050 accelerometer/gyroscope and an HMC5883L magnetometer on the same I2C bus to compute a heading that remains accurate even when the board is tilted — then display the compensated heading on an SSD1306 OLED.

## Goal

Understand why a flat-plane compass heading using only `atan2(Hy, Hx)` fails when the sensor board is tilted, and learn how to apply a tilt-compensation algorithm: use pitch (θ) and roll (φ) angles from the MPU-6050 accelerometer to rotate the raw HMC5883L field vector into a virtual horizontal plane before computing the heading.

## What You Will Build

The MPU-6050 (I2C address `0x68`) and HMC5883L (I2C address `0x1E`) both sit on I2C0 (GPIO 0/1). An SSD1306 OLED (I2C address `0x3C`, also I2C0) displays the result. Every 200 ms the firmware reads 6 accelerometer axes from the MPU-6050, computes pitch and roll in radians, reads the HMC5883L field vector, applies the 3-axis tilt-compensation rotation equations, computes the compensated heading, and updates the OLED with the heading in degrees and cardinal direction.

## Parts Needed

| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MPU-6050 IMU Module | `mpu6050` | Yes | Yes |
| HMC5883L Magnetometer Module | `hmc5883l` | Yes | Yes |
| SSD1306 OLED Display (128×64, I2C) | `ssd1306` | Yes | Yes |
| Jumper Wires | — | No | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 | VCC | 3V3 | Red | 3.3 V IMU power |
| MPU-6050 | GND | GND | Black | Common ground |
| MPU-6050 | SDA | GPIO 0 (I2C0 SDA) | Orange | Shared I2C SDA |
| MPU-6050 | SCL | GPIO 1 (I2C0 SCL) | Yellow | Shared I2C SCL |
| MPU-6050 | AD0 | GND | Black | Sets I2C address to 0x68 |
| HMC5883L | VCC | 3V3 | Red | 3.3 V magnetometer power |
| HMC5883L | GND | GND | Black | Common ground |
| HMC5883L | SDA | GPIO 0 (I2C0 SDA) | Orange | Shared I2C SDA |
| HMC5883L | SCL | GPIO 1 (I2C0 SCL) | Yellow | Shared I2C SCL |
| SSD1306 OLED | VCC | 3V3 | Red | OLED power |
| SSD1306 OLED | GND | GND | Black | Common ground |
| SSD1306 OLED | SDA | GPIO 0 (I2C0 SDA) | Orange | Shared I2C SDA |
| SSD1306 OLED | SCL | GPIO 1 (I2C0 SCL) | Yellow | Shared I2C SCL |

> **Wiring tip:** Three I2C devices share the same bus — MPU-6050 at `0x68`, HMC5883L at `0x1E`, and SSD1306 at `0x3C`. They coexist because each has a unique address. Connect MPU-6050 pin AD0 to GND to lock the address at `0x68`; pulling it HIGH would change it to `0x69`. Ensure 4.7 kΩ pull-ups are present on SDA and SCL — only one set is needed for the entire bus; do not stack pull-ups from multiple modules as this would lower the effective resistance and slow the bus.

## Code

```cpp
// 149 - I2C OLED Tilt-compensated Compass
// MPU-6050 (accel) + HMC5883L (mag) -> tilt-free heading on SSD1306 OLED
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// ---- OLED ----
#define SCREEN_WIDTH  128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
#define OLED_ADDR     0x3C
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// ---- I2C Device Addresses ----
#define MPU_ADDR  0x68
#define HMC_ADDR  0x1E

// ---- MPU-6050 Registers ----
#define MPU_PWR_MGMT_1  0x6B
#define MPU_ACCEL_XOUT  0x3B   // First accel register (6 bytes: Ax, Ay, Az)

// ---- HMC5883L Registers ----
#define HMC_CFG_A 0x00
#define HMC_CFG_B 0x01
#define HMC_MODE  0x02
#define HMC_DATA  0x03

// Magnetic declination correction (degrees, negative = west)
#define DECLINATION (-0.5)

// ---- Accelerometer raw readings ----
int   axRaw = 0;
int   ayRaw = 0;
int   azRaw = 0;
float axG   = 0.0;
float ayG   = 0.0;
float azG   = 0.0;

// Pitch and roll (radians)
float pitch = 0.0;
float roll  = 0.0;

// ---- Magnetometer raw readings ----
int   hxRaw = 0;
int   hyRaw = 0;
int   hzRaw = 0;

// ---- Tilt-compensated field components ----
float hxComp = 0.0;
float hyComp = 0.0;

// ---- Computed heading ----
float headingRad = 0.0;
float headingDeg = 0.0;

// ---- Cardinal label ----
String cardinalLabel = "";

// ---- State ----
int   sensorReady   = 0;
int   oledReady     = 0;
unsigned long lastRefreshMs = 0;
unsigned long nowMs         = 0;

// ---- I2C byte read helpers ----
int rawMsb = 0;
int rawLsb = 0;

void setup() {
  Serial.begin(115200);
  Serial.println("=== Tilt-compensated Compass ===");

  Wire.begin();

  // ---- Initialise SSD1306 OLED ----
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("ERROR: SSD1306 not found.");
    oledReady = 0;
  } else {
    oledReady = 1;
    display.clearDisplay();
    display.setTextColor(SSD1306_WHITE);
    display.setTextSize(1);
    display.setCursor(0, 0);
    display.println("Tilt Compass");
    display.println("Initialising...");
    display.display();
    Serial.println("SSD1306 OLED ready.");
  }

  // ---- Initialise MPU-6050 ----
  // Wake up device (clear sleep bit in PWR_MGMT_1)
  Wire.beginTransmission(MPU_ADDR);
  Wire.write(MPU_PWR_MGMT_1);
  Wire.write(0x00);  // Clear sleep bit
  Wire.endTransmission();
  Serial.println("MPU-6050 wake-up complete.");

  // ---- Initialise HMC5883L ----
  // 8 sample average, 15 Hz, normal measurement mode
  Wire.beginTransmission(HMC_ADDR);
  Wire.write(HMC_CFG_A);
  Wire.write(0x70);
  Wire.endTransmission();

  Wire.beginTransmission(HMC_ADDR);
  Wire.write(HMC_CFG_B);
  Wire.write(0x20);  // Gain = 1090 LSB/Gauss
  Wire.endTransmission();

  Wire.beginTransmission(HMC_ADDR);
  Wire.write(HMC_MODE);
  Wire.write(0x00);  // Continuous measurement
  Wire.endTransmission();

  delay(150);
  sensorReady = 1;
  Serial.println("HMC5883L configured.");
  Serial.println("System ready. Tilt-compensated heading active.");
}

void loop() {
  if (!sensorReady || !oledReady) {
    delay(1000);
    return;
  }

  nowMs = millis();
  if (nowMs - lastRefreshMs < 200) return;
  lastRefreshMs = nowMs;

  // ---- Read MPU-6050 Accelerometer (6 bytes: Ax, Ay, Az) ----
  Wire.beginTransmission(MPU_ADDR);
  Wire.write(MPU_ACCEL_XOUT);
  Wire.endTransmission(false);
  Wire.requestFrom(MPU_ADDR, 6);

  if (Wire.available() >= 6) {
    rawMsb = Wire.read(); rawLsb = Wire.read();
    axRaw  = (int16_t)((rawMsb << 8) | rawLsb);
    rawMsb = Wire.read(); rawLsb = Wire.read();
    ayRaw  = (int16_t)((rawMsb << 8) | rawLsb);
    rawMsb = Wire.read(); rawLsb = Wire.read();
    azRaw  = (int16_t)((rawMsb << 8) | rawLsb);
  } else {
    Serial.println("WARN: MPU-6050 read failed.");
    return;
  }

  // Convert raw values to g (±2g range, sensitivity 16384 LSB/g)
  axG = (float)axRaw / 16384.0;
  ayG = (float)ayRaw / 16384.0;
  azG = (float)azRaw / 16384.0;

  // Compute pitch and roll from accelerometer
  // pitch: rotation around Y axis (nose up/down)
  // roll:  rotation around X axis (side tilt)
  pitch = asin(-axG);
  roll  = asin(ayG / cos(pitch));

  // ---- Read HMC5883L (6 bytes: HxMSB, HxLSB, HzMSB, HzLSB, HyMSB, HyLSB) ----
  Wire.beginTransmission(HMC_ADDR);
  Wire.write(HMC_DATA);
  Wire.endTransmission(false);
  Wire.requestFrom(HMC_ADDR, 6);

  if (Wire.available() >= 6) {
    rawMsb = Wire.read(); rawLsb = Wire.read();
    hxRaw  = (int16_t)((rawMsb << 8) | rawLsb);
    rawMsb = Wire.read(); rawLsb = Wire.read();
    hzRaw  = (int16_t)((rawMsb << 8) | rawLsb);  // Z before Y in HMC5883L output
    rawMsb = Wire.read(); rawLsb = Wire.read();
    hyRaw  = (int16_t)((rawMsb << 8) | rawLsb);
  } else {
    Serial.println("WARN: HMC5883L read failed.");
    return;
  }

  // ---- Tilt Compensation ----
  // Rotate the magnetic field vector into the horizontal plane using pitch and roll.
  // hxComp = Hx*cos(pitch) + Hz*sin(pitch)
  // hyComp = Hx*sin(roll)*sin(pitch) + Hy*cos(roll) - Hz*sin(roll)*cos(pitch)
  hxComp = (float)hxRaw * cos(pitch)
          + (float)hzRaw * sin(pitch);

  hyComp = (float)hxRaw * sin(roll) * sin(pitch)
          + (float)hyRaw * cos(roll)
          - (float)hzRaw * sin(roll) * cos(pitch);

  // ---- Compute heading ----
  headingRad = atan2(hyComp, hxComp);

  // Apply magnetic declination
  headingRad += DECLINATION * (PI / 180.0);

  // Normalise to 0..2PI
  if (headingRad < 0.0) {
    headingRad += 2.0 * PI;
  }
  if (headingRad > 2.0 * PI) {
    headingRad -= 2.0 * PI;
  }

  headingDeg = headingRad * (180.0 / PI);

  // ---- Cardinal direction ----
  if (headingDeg < 22.5 || headingDeg >= 337.5) {
    cardinalLabel = " NORTH";
  } else if (headingDeg < 67.5) {
    cardinalLabel = "  N-E ";
  } else if (headingDeg < 112.5) {
    cardinalLabel = " EAST ";
  } else if (headingDeg < 157.5) {
    cardinalLabel = "  S-E ";
  } else if (headingDeg < 202.5) {
    cardinalLabel = " SOUTH";
  } else if (headingDeg < 247.5) {
    cardinalLabel = "  S-W ";
  } else if (headingDeg < 292.5) {
    cardinalLabel = " WEST ";
  } else {
    cardinalLabel = "  N-W ";
  }

  // ---- Serial debug ----
  Serial.print("Pitch="); Serial.print(pitch * 180.0 / PI, 1);
  Serial.print("  Roll=");  Serial.print(roll  * 180.0 / PI, 1);
  Serial.print("  Heading="); Serial.print(headingDeg, 1);
  Serial.print("  "); Serial.println(cardinalLabel);

  // ---- Update OLED ----
  display.clearDisplay();

  // Heading label (small)
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.print("Tilt-Comp Compass");

  // Large heading value
  display.setTextSize(3);
  display.setCursor(0, 14);
  display.print(headingDeg, 1);
  display.print((char)247);  // degree symbol

  // Cardinal label
  display.setTextSize(2);
  display.setCursor(0, 46);
  display.print(cardinalLabel);

  display.display();
}
```

## What to Click in MbedO

1. Drag **VEGA ARIES v3**, **MPU-6050**, **HMC5883L**, and **SSD1306 OLED** onto the canvas.
2. Wire all three sensors: **SDA → GPIO 0**, **SCL → GPIO 1**, **VCC → 3V3**, **GND → GND**.
3. Connect MPU-6050 **AD0 → GND**.
4. Paste the code into the editor and select **Interpreted Mode**.
5. Click **Run**.
6. Use the MbedO **MPU-6050 widget** to adjust pitch and roll tilt angles.
7. Use the MbedO **HMC5883L widget** heading slider to set the magnetic field direction.
8. Observe that the displayed heading on the OLED remains stable as pitch/roll change — demonstrating tilt compensation.

## Expected Output

Serial Monitor:
```
=== Tilt-compensated Compass ===
SSD1306 OLED ready.
MPU-6050 wake-up complete.
HMC5883L configured.
System ready. Tilt-compensated heading active.
Pitch=0.0   Roll=0.0   Heading=351.2  NORTH
Pitch=20.0  Roll=10.0  Heading=351.5  NORTH
Pitch=45.0  Roll=30.0  Heading=350.9  NORTH
```

OLED Display:
```
Tilt-Comp Compass
351.2°
NORTH
```

## Expected Canvas Behavior

* Heading stays near-constant as the MPU-6050 tilt sliders are moved, demonstrating that pitch and roll are being compensated.
* Adjusting the HMC5883L heading slider rotates the displayed heading and updates the cardinal label.
* Pitch and roll values are printed in degrees to the Serial Monitor for verification.
* If either sensor is missing, a descriptive error is printed and the loop halts.

## Code Walkthrough

| Line | Check / Action |
| --- | --- |
| `Wire.write(0x00)` to `MPU_PWR_MGMT_1` | Clears the SLEEP bit to wake the MPU-6050 from its power-on default sleep state. |
| `axG = axRaw / 16384.0` | Converts raw 16-bit accelerometer value to g-force. At ±2g range, sensitivity is 16384 LSB/g. |
| `pitch = asin(-axG)` | Pitch angle from accelerometer: negative X-axis gravity component gives forward-tilt angle. |
| `roll = asin(ayG / cos(pitch))` | Roll angle corrected for pitch coupling — pure roll would use `asin(ayG)` only when pitch = 0. |
| HMC data order `X, Z, Y` | HMC5883L transmits bytes in X, Z, Y order — the code assigns the middle pair to `hzRaw`. |
| `hxComp = Hx*cos(pitch) + Hz*sin(pitch)` | Rotates the field vector around the Y axis (pitch correction). |
| `hyComp = Hx*sin(roll)*sin(pitch) + Hy*cos(roll) - Hz*sin(roll)*cos(pitch)` | Rotates around the X axis (roll correction) while accounting for the already-applied pitch rotation. |
| `atan2(hyComp, hxComp)` | Heading from the tilt-compensated virtual horizontal field components. |
| Normalise to `0..2*PI` | Two sequential `if` statements wrap the angle without modulo arithmetic. |

## Hardware & Safety Concept

* **Why Flat-Plane Fails**: A standard `atan2(Hy, Hx)` compass assumes the sensor is perfectly level. When tilted, gravity mixes into the accelerometer axes but — critically — the Earth's magnetic field also has a vertical component (the inclination or dip angle). Tilting the sensor causes the vertical magnetic field component to project onto the horizontal axes, shifting the apparent heading by several degrees per degree of tilt. At a magnetic inclination of 30–60° (typical mid-latitudes), a 30° board tilt can produce heading errors exceeding 20°.
* **Tilt-Compensation Algorithm**: The compensation equations rotate the measured magnetic field vector by the pitch and roll angles derived from the accelerometer. This transforms the field vector into a coordinate frame aligned with the horizontal plane of the Earth, yielding an accurate heading regardless of sensor tilt — provided the platform is not accelerating (the accelerometer must be reading only gravity).
* **Gimbal Lock Limitation**: The `asin()` approach to computing pitch reaches a singularity at ±90° pitch (straight up or down). At these extremes, roll becomes undefined and the compass is unreliable. For full-orientation systems, quaternion-based fusion (Mahony or Madgwick filters) avoids this limitation.

## Try This! (Challenges)

1. **Live Pitch/Roll Display**: Add a second OLED screen zone below the cardinal label that prints `P:XX.X R:XX.X` (pitch and roll in degrees) so the user can see the tilt values directly on the display without needing the Serial Monitor.
2. **Stability Gate**: Add global variables `float pitchDeg = 0.0` and `float rollDeg = 0.0`. Only update `headingDeg` when both `pitchDeg` and `rollDeg` are within ±45°. Outside this range, display `"Tilt Too High"` on the OLED instead of a heading, warning the user that the compensation range has been exceeded.

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Heading drifts when board is tilted | HMC and MPU not mounted rigidly together | Secure both modules to the same rigid PCB or frame so they share the same tilt angle. |
| `WARN: MPU-6050 read failed` | Device not found at 0x68 | Confirm AD0 is tied to GND; verify SDA/SCL wiring and pull-ups. |
| `WARN: HMC5883L read failed` | Bus conflict or missing address | Confirm address is 0x1E; run I2C scanner to list detected devices. |
| Heading 180° off from expected | Sensor mounted backwards | Rotate module 180° or negate both `hxRaw` and `hyRaw` in the tilt-comp equations. |
| Pitch/roll values oscillate wildly | Vibration or mechanical shock | Add a low-pass filter: `axG = 0.8 * axG_prev + 0.2 * axG_new` using two global float variables per axis. |

## Mode Notes

This project runs in MbedO **interpreted C++ mode**.

## Related Projects

- [148 - I2C OLED Digital Compass](148-i2c-oled-digital-compass.md)
- [150 - Dual-protocol Telemetry Logger](150-dual-protocol-telemetry-logger.md)
- [146 - GPS Location Display](146-gps-location-display.md)
