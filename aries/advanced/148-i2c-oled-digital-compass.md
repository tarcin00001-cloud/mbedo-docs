# 148 - I2C OLED Digital Compass

Read magnetic-field vector data from an HMC5883L triple-axis magnetometer over I2C and display the computed heading angle on an SSD1306 OLED panel as both a numeric degree readout and a large cardinal-direction label.

## Goal

Learn how to communicate with the HMC5883L magnetometer via I2C, compute a compass heading angle from the raw X and Y magnetic-field components using `atan2()`, apply a fixed magnetic declination correction, and render the result on a 128×64 SSD1306 OLED display — all within MbedO's interpreted C++ mode constraints.

## What You Will Build

The HMC5883L sits on the I2C0 bus (GPIO 0/1). The firmware configures the sensor's measurement mode and data rate registers at startup, then reads three 16-bit signed integers (Hx, Hy, Hz) every 200 ms. It computes heading = `atan2(Hy, Hx)` in degrees, maps it to 0–360°, and classifies it into one of eight cardinal/intercardinal directions. The SSD1306 OLED (also on I2C0 at address 0x3C) shows the heading in large text on line 1 and the cardinal label on line 2, refreshing at 5 Hz.

## Parts Needed

| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| HMC5883L Magnetometer Module | `hmc5883l` | Yes | Yes |
| SSD1306 OLED Display (128×64, I2C) | `ssd1306` | Yes | Yes |
| Jumper Wires | — | No | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HMC5883L | VCC | 3V3 | Red | 3.3 V magnetometer power |
| HMC5883L | GND | GND | Black | Common ground |
| HMC5883L | SDA | GPIO 0 (I2C0 SDA) | Orange | I2C data — shared with OLED |
| HMC5883L | SCL | GPIO 1 (I2C0 SCL) | Yellow | I2C clock — shared with OLED |
| SSD1306 OLED | VCC | 3V3 | Red | OLED power |
| SSD1306 OLED | GND | GND | Black | Common ground |
| SSD1306 OLED | SDA | GPIO 0 (I2C0 SDA) | Orange | Shared I2C SDA |
| SSD1306 OLED | SCL | GPIO 1 (I2C0 SCL) | Yellow | Shared I2C SCL |

> **Wiring tip:** Both the HMC5883L (default I2C address `0x1E`) and SSD1306 OLED (default address `0x3C`) share the same I2C0 bus. They coexist without conflict because they have different addresses. Ensure I2C pull-up resistors (4.7 kΩ to 3V3) are present on SDA and SCL — most HMC5883L breakout modules include them. Keep the compass module away from power cables, motors, or ferrous metal objects to avoid magnetic interference.

## Code

```cpp
// 148 - I2C OLED Digital Compass
// HMC5883L magnetometer -> heading on SSD1306 OLED
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// OLED configuration
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
#define OLED_ADDRESS  0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// HMC5883L I2C address and register addresses
#define HMC_ADDR      0x1E
#define HMC_REG_CFG_A 0x00
#define HMC_REG_CFG_B 0x01
#define HMC_REG_MODE  0x02
#define HMC_REG_DATA  0x03  // First data register (Hx MSB)

// Magnetic declination correction in degrees
// Positive = East, Negative = West
// Example: Bengaluru, India ~ -0.5 degrees
#define DECLINATION_DEG (-0.5)

// Raw magnetometer readings (16-bit signed)
int   hxRaw = 0;
int   hyRaw = 0;
int   hzRaw = 0;

// Computed heading
float headingRad = 0.0;
float headingDeg = 0.0;

// Cardinal direction label
String cardinalLabel = "";

// Display refresh timer
unsigned long lastRefreshMs = 0;
unsigned long nowMs         = 0;

// I2C read helpers (state variables)
int    rawMsb = 0;
int    rawLsb = 0;
int    rawVal = 0;

// Sensor init success flag
int sensorReady = 0;
int oledReady   = 0;

void setup() {
  Serial.begin(115200);
  Serial.println("=== I2C OLED Digital Compass ===");

  Wire.begin();

  // Initialise SSD1306 OLED
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDRESS)) {
    Serial.println("ERROR: SSD1306 OLED not found. Check I2C address.");
    oledReady = 0;
  } else {
    oledReady = 1;
    display.clearDisplay();
    display.setTextColor(SSD1306_WHITE);
    display.setTextSize(1);
    display.setCursor(0, 0);
    display.println("Digital Compass");
    display.println("Init HMC5883L...");
    display.display();
    Serial.println("SSD1306 OLED ready.");
  }

  // Configure HMC5883L
  // Config Register A: 8 samples averaged, 15 Hz output, normal measurement
  Wire.beginTransmission(HMC_ADDR);
  Wire.write(HMC_REG_CFG_A);
  Wire.write(0x70);  // 0b01110000 = 8 avg, 15 Hz, normal
  Wire.endTransmission();

  // Config Register B: Gain = 1090 LSB/Gauss (default)
  Wire.beginTransmission(HMC_ADDR);
  Wire.write(HMC_REG_CFG_B);
  Wire.write(0x20);
  Wire.endTransmission();

  // Mode Register: continuous measurement mode
  Wire.beginTransmission(HMC_ADDR);
  Wire.write(HMC_REG_MODE);
  Wire.write(0x00);  // Continuous measurement
  Wire.endTransmission();

  delay(100);  // Allow first measurement to complete
  sensorReady = 1;
  Serial.println("HMC5883L configured. Continuous measurement mode.");
}

void loop() {
  if (!sensorReady || !oledReady) {
    delay(1000);
    return;
  }

  nowMs = millis();
  if (nowMs - lastRefreshMs < 200) return;
  lastRefreshMs = nowMs;

  // ---- Read 6 bytes of raw field data from HMC5883L ----
  // Register layout: HxMSB, HxLSB, HzMSB, HzLSB, HyMSB, HyLSB
  Wire.beginTransmission(HMC_ADDR);
  Wire.write(HMC_REG_DATA);
  Wire.endTransmission(false);  // Repeated start
  Wire.requestFrom(HMC_ADDR, 6);

  if (Wire.available() >= 6) {
    // Hx
    rawMsb = Wire.read();
    rawLsb = Wire.read();
    rawVal = (int16_t)((rawMsb << 8) | rawLsb);
    hxRaw  = rawVal;

    // Hz (note: HMC5883L data order is X, Z, Y)
    rawMsb = Wire.read();
    rawLsb = Wire.read();
    rawVal = (int16_t)((rawMsb << 8) | rawLsb);
    hzRaw  = rawVal;

    // Hy
    rawMsb = Wire.read();
    rawLsb = Wire.read();
    rawVal = (int16_t)((rawMsb << 8) | rawLsb);
    hyRaw  = rawVal;
  } else {
    Serial.println("WARN: Insufficient bytes from HMC5883L.");
    return;
  }

  // ---- Compute heading from Hx and Hy ----
  headingRad = atan2((float)hyRaw, (float)hxRaw);

  // Apply magnetic declination
  headingRad += DECLINATION_DEG * (PI / 180.0);

  // Normalise to 0..2PI
  if (headingRad < 0) {
    headingRad += 2.0 * PI;
  }
  if (headingRad > 2.0 * PI) {
    headingRad -= 2.0 * PI;
  }

  headingDeg = headingRad * (180.0 / PI);

  // ---- Map to cardinal direction ----
  // 8 sectors of 45 degrees each, offset by 22.5 degrees
  if (headingDeg < 22.5 || headingDeg >= 337.5) {
    cardinalLabel = "  NORTH";
  } else if (headingDeg < 67.5) {
    cardinalLabel = "   N-E";
  } else if (headingDeg < 112.5) {
    cardinalLabel = "  EAST";
  } else if (headingDeg < 157.5) {
    cardinalLabel = "   S-E";
  } else if (headingDeg < 202.5) {
    cardinalLabel = "  SOUTH";
  } else if (headingDeg < 247.5) {
    cardinalLabel = "   S-W";
  } else if (headingDeg < 292.5) {
    cardinalLabel = "  WEST";
  } else {
    cardinalLabel = "   N-W";
  }

  // ---- Debug output ----
  Serial.print("Hx="); Serial.print(hxRaw);
  Serial.print("  Hy="); Serial.print(hyRaw);
  Serial.print("  Heading="); Serial.print(headingDeg, 1);
  Serial.print("  "); Serial.println(cardinalLabel);

  // ---- Update OLED ----
  display.clearDisplay();

  // Large heading value — text size 3 = ~18px tall
  display.setTextSize(3);
  display.setCursor(10, 4);
  display.print(headingDeg, 1);
  display.print((char)247);  // degree symbol

  // Cardinal direction in medium text
  display.setTextSize(2);
  display.setCursor(0, 44);
  display.print(cardinalLabel);

  display.display();
}
```

## What to Click in MbedO

1. Drag **VEGA ARIES v3**, **HMC5883L**, and **SSD1306 OLED** components onto the canvas.
2. Wire both HMC5883L and SSD1306: **SDA → GPIO 0**, **SCL → GPIO 1**, **VCC → 3V3**, **GND → GND**.
3. Paste the code into the editor and select **Interpreted Mode**.
4. Click **Run**.
5. Use the HMC5883L widget's **Heading** slider to rotate the simulated compass bearing.
6. Observe the numeric heading and cardinal label updating on the OLED canvas widget.

## Expected Output

Serial Monitor:
```
=== I2C OLED Digital Compass ===
SSD1306 OLED ready.
HMC5883L configured. Continuous measurement mode.
Hx=312  Hy=-47  Heading=351.5  NORTH
Hx=312  Hy=-47  Heading=351.5  NORTH
Hx=200  Hy=200  Heading=44.8    N-E
```

OLED Display:
```
Line 1 (large):  351.5°
Line 2 (medium): NORTH
```

## Expected Canvas Behavior

* The OLED refreshes at 5 Hz (every 200 ms).
* As the HMC5883L widget heading slider is adjusted, the numeric degree value and cardinal label update on the next refresh cycle.
* If the sensor or OLED is not detected at startup, the error is printed to Serial and the `loop()` halts gracefully.
* The heading wraps correctly from 359.9° back to 0.0° (North).

## Code Walkthrough

| Line | Check / Action |
| --- | --- |
| `Wire.write(0x70)` to `HMC_REG_CFG_A` | Sets 8-sample averaging and 15 Hz output rate for the magnetometer. |
| `Wire.write(0x00)` to `HMC_REG_MODE` | Places sensor in continuous measurement mode (polls every ~67 ms internally). |
| `Wire.requestFrom(HMC_ADDR, 6)` | Requests 6 bytes: HxMSB, HxLSB, HzMSB, HzLSB, HyMSB, HyLSB in that order. |
| `(int16_t)((rawMsb << 8) \| rawLsb)` | Reconstructs a signed 16-bit integer from two bytes (big-endian as transmitted by HMC5883L). |
| `atan2(hyRaw, hxRaw)` | Returns the angle in radians from the positive X axis to the vector (Hx, Hy). |
| `headingRad += DECLINATION_DEG * (PI / 180.0)` | Corrects for local magnetic declination to give true north. |
| Normalise to `0..2*PI` | Two separate `if` statements handle both underflow (negative) and overflow (> 2π). |
| `headingDeg < 22.5 \|\| >= 337.5` | North straddles 0°; the OR condition captures both the high end and low end of the 360° circle. |

## Hardware & Safety Concept

* **Magnetic Declination**: True North (geographic) and Magnetic North differ by a location-dependent angle called the magnetic declination. In Bengaluru it is approximately −0.5° (west). Ignoring this causes a fixed offset error in compass heading. The `DECLINATION_DEG` constant must be updated for each deployment location using resources such as NOAA's magnetic declination calculator.
* **HMC5883L Register Map**: The sensor exposes three consecutive 16-bit registers. Importantly, the output order is **X, Z, Y** (not X, Y, Z). Reading them in the wrong order would swap the Z and Y fields and produce incorrect headings. The firmware explicitly assigns the second two-byte pair to `hzRaw` and the third to `hyRaw`.
* **Magnetic Interference**: Ferrous metals (screws, chassis), permanent magnets (speakers, motors), and current-carrying wires all distort the local magnetic field sensed by the HMC5883L. For best accuracy, mount the module away from power traces and metal enclosures, and perform a hard-iron calibration by rotating the board through 360° and recording min/max values for each axis.

## Try This! (Challenges)

1. **Heading Change Alert**: Add a global `float prevHeadingDeg = -1.0` variable. After computing `headingDeg`, compare it to the previous reading. If the difference exceeds 10 degrees, print `"SIGNIFICANT TURN: X.X deg"` to the Serial Monitor, then update `prevHeadingDeg`.
2. **Declination Configurability**: Replace the `#define DECLINATION_DEG` with a global `float declinationDeg = -0.5`. Print the applied declination value to the OLED on startup so the operator can verify the calibration constant in use.

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| `ERROR: SSD1306 OLED not found` | Wrong I2C address | Try address `0x3D` instead of `0x3C`. Run I2C scanner to find the actual address. |
| Heading stuck at 0° or NaN | HMC5883L not responding | Check SDA/SCL wiring; confirm module power at 3.3 V; verify pull-up resistors are present. |
| Heading jitters ±10–20° constantly | Magnetic interference nearby | Move module away from USB cables, buck converters, and metal surfaces. |
| Cardinal direction wrong by ~180° | Module mounted upside-down | Flip the module or negate both `hxRaw` and `hyRaw` in code to correct orientation. |
| OLED shows garbled pixels | I2C address conflict | HMC5883L at `0x1E` and SSD1306 at `0x3C` should not conflict; confirm no other I2C device on the bus. |

## Mode Notes

This project runs in MbedO **interpreted C++ mode**.

## Related Projects

- [149 - I2C OLED Tilt-compensated Compass](149-i2c-oled-tilt-compensated-compass.md)
- [146 - GPS Location Display](146-gps-location-display.md)
- [150 - Dual-protocol Telemetry Logger](150-dual-protocol-telemetry-logger.md)
