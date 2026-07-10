# 108 - Digital Spirit Level

Use an MPU-6050 IMU to read raw acceleration on the X and Z axes and display a human-readable tilt label on a 16x2 I2C LCD — no calibration library required.

## Goal
Learn how to read raw accelerometer registers from the MPU-6050 over I2C, scale the values into gravitational units, and map axis readings to plain-English orientation labels displayed on an LCD.

## What You Will Build
The sketch reads Ac_x and Ac_z from the MPU-6050 every 300 ms and decides the current tilt orientation:

- **LEVEL**: Both axes near zero — the board is flat.
- **TILT LEFT / TILT RIGHT**: Ac_x is strongly negative or positive.
- **TILT FORWARD / TILT BACK**: Ac_z is strongly negative or positive.

The tilt label and the raw Ac_x / Ac_z values print on the LCD and in the Serial Terminal simultaneously.

**Why A4/A5 and the MPU-6050 address?** The Arduino Uno's hardware I2C bus is on A4 (SDA) and A5 (SCL). The MPU-6050 always responds at address `0x68` (AD0 pulled low). The LCD backpack sits at `0x27` — both devices share the same two-wire bus safely because they have different addresses.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| MPU-6050 IMU | `mpu6050` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| MPU-6050 | VCC | 3.3V | Power supply (some modules also accept 5V) |
| MPU-6050 | SDA | A4 | I2C Data bus |
| MPU-6050 | SCL | A5 | I2C Clock bus |
| MPU-6050 | GND | GND | Ground reference |
| MPU-6050 | AD0 | GND | Sets I2C address to 0x68 |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data bus (shared) |
| 16x2 I2C LCD | SCL | A5 | I2C Clock bus (shared) |
| 16x2 I2C LCD | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int MPU_ADDR = 0x68;

LiquidCrystal_I2C lcd(0x27, 16, 2);

int16_t Ac_x = 0;
int16_t Ac_y = 0;
int16_t Ac_z = 0;

void setup() {
  Serial.begin(9600);
  Wire.begin();

  // Wake the MPU-6050 (clear sleep bit in PWR_MGMT_1)
  Wire.beginTransmission(MPU_ADDR);
  Wire.write(0x6B);
  Wire.write(0x00);
  Wire.endTransmission(true);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Spirit Level");
  lcd.setCursor(0, 1);
  lcd.print("Initializing...");
  delay(1500);
  lcd.clear();

  Serial.println("Digital Spirit Level Ready");
}

void loop() {
  // Request 6 bytes of accelerometer data starting at register 0x3B
  Wire.beginTransmission(MPU_ADDR);
  Wire.write(0x3B);
  Wire.endTransmission(false);
  Wire.requestFrom(MPU_ADDR, 6, true);

  Ac_x = (Wire.read() << 8) | Wire.read();
  Ac_y = (Wire.read() << 8) | Wire.read();
  Ac_z = (Wire.read() << 8) | Wire.read();

  // Scale to g (default full-scale +/-2g, 16384 LSB/g)
  float gx = Ac_x / 16384.0;
  float gz = Ac_z / 16384.0;

  // Determine tilt label from axis dominance
  String tiltLabel = "LEVEL";

  if (gx > 0.4) {
    tiltLabel = "TILT RIGHT";
  } else if (gx < -0.4) {
    tiltLabel = "TILT LEFT";
  } else if (gz > 0.4) {
    tiltLabel = "TILT BACK";
  } else if (gz < -0.4) {
    tiltLabel = "TILT FORWARD";
  }

  // Serial output
  Serial.print("Ac_x: "); Serial.print(gx, 2);
  Serial.print(" g  Ac_z: "); Serial.print(gz, 2);
  Serial.print(" g  -> "); Serial.println(tiltLabel);

  // LCD line 1: tilt label (trailing spaces clear stale characters)
  lcd.setCursor(0, 0);
  lcd.print(tiltLabel);
  lcd.print("          ");

  // LCD line 2: raw g values
  lcd.setCursor(0, 1);
  lcd.print("X:");
  lcd.print(gx, 1);
  lcd.print(" Z:");
  lcd.print(gz, 1);
  lcd.print("   ");

  delay(300);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **MPU-6050**, and **16x2 I2C LCD** onto the canvas.
2. Connect MPU-6050: **VCC** to **3.3V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**, **AD0** to **GND**.
3. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Double-click the MPU-6050 on the canvas and adjust the X / Z tilt sliders to watch the label change on both the LCD and the Terminal.

## Expected Output

Terminal:
```
Digital Spirit Level Ready
Ac_x: 0.02 g  Ac_z: 0.01 g  -> LEVEL
Ac_x: 0.65 g  Ac_z: 0.02 g  -> TILT RIGHT
Ac_x: -0.70 g  Ac_z: 0.01 g  -> TILT LEFT
Ac_x: 0.02 g  Ac_z: 0.55 g  -> TILT BACK
```

LCD Display:
```
TILT RIGHT
X:0.6 Z:0.0
```

## Expected Canvas Behavior
| Ac_x (g) | Ac_z (g) | LCD Line 1 | LCD Line 2 |
| --- | --- | --- | --- |
| 0.00 | 0.00 | `LEVEL` | `X:0.0 Z:0.0` |
| +0.65 | 0.00 | `TILT RIGHT` | `X:0.6 Z:0.0` |
| -0.70 | 0.00 | `TILT LEFT` | `X:-0.7 Z:0.0` |
| 0.00 | +0.55 | `TILT BACK` | `X:0.0 Z:0.5` |
| 0.00 | -0.60 | `TILT FORWARD` | `X:0.0 Z:-0.6` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Wire.write(0x6B); Wire.write(0x00)` | Writes 0 to the PWR_MGMT_1 register, clearing the sleep bit and waking the MPU-6050. |
| `(Wire.read() << 8) \| Wire.read()` | Combines two successive I2C bytes into one signed 16-bit integer (big-endian). |
| `Ac_x / 16384.0` | Converts raw ADC count to gravitational units. At default ±2 g full-scale, 1 g = 16 384 LSB. |
| `gx > 0.4` threshold | A 0.4 g dead-band ignores minor vibration; only a deliberate ~24-degree tilt changes the label. |
| Trailing `"          "` spaces | Erases stale LCD characters when a longer label (e.g., `TILT FORWARD`) is replaced by a shorter one (e.g., `LEVEL`). |

## Hardware & Safety Concept: I2C Multi-Device Addressing
The MPU-6050 (address `0x68`) and the LCD backpack (address `0x27`) share the same two SDA/SCL wires without conflict because each device has a unique 7-bit address. The Arduino acts as the I2C bus master and addresses each device individually before every transaction. Never connect two I2C devices with identical addresses on the same bus — their simultaneous responses will corrupt each other's data.

## Try This! (Challenges)
1. **Angle in Degrees**: Compute `atan2(gx, 1.0) * 180.0 / PI` and display the numeric tilt angle on LCD line 2 instead of the raw g value.
2. **Bubble Bargraph**: Use `map()` to convert gx (-1.0 to +1.0) to a column position (0 to 15) and print a `*` character at that position on LCD line 1 to simulate a bubble-tube level indicator.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD stays blank | I2C address mismatch | Try changing `0x27` to `0x3F` in the `LiquidCrystal_I2C` constructor. |
| Ac_x / Ac_z always read 0 | MPU-6050 still in sleep mode | Verify the `Wire.write(0x6B); Wire.write(0x00)` wake-up sequence runs in `setup()`. |
| Label flickers rapidly | Threshold too low | Increase the `0.4` threshold to `0.6` to add a larger dead-band. |

## Mode Notes
Raw Wire I2C register reads, `String` label logic, and `LiquidCrystal_I2C` output are all fully supported by MbedO interpreted mode.

## Related Projects
- [86 - Weather Station](86-weather-station.md)
- [61 - Weather Readout LCD](../intermediate/61-weather-readout-lcd.md)
- [51 - Temp Display LCD](../intermediate/51-temp-display-lcd.md)
- [115 - Servo Radar](115-servo-radar.md)
