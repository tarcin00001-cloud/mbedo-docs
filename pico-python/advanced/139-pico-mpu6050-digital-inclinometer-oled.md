# 139 - Pico MPU-6050 Digital Inclinometer OLED

Build a precision digital inclinometer that reads pitch and roll angles from an MPU-6050 IMU using raw accelerometer data, calculates tilt angles in degrees, and displays a live graphical level bubble on an SSD1306 OLED.

## Goal
Learn how to read raw accelerometer data from an MPU-6050 over I2C, calculate pitch and roll angles using arctangent trigonometry, and render a graphical bubble-level display on an SSD1306 OLED in MicroPython.

## What You Will Build
A digital inclinometer:
- **MPU-6050 IMU (GP4, GP5)**: Reads X/Y/Z accelerometer axes.
- **SSD1306 OLED (GP4, GP5 I2C Bus 0)**: Shared I2C bus — displays a graphical bubble level and numeric pitch/roll angles.
- **Buzzer (GP15)**: Sounds when tilt exceeds a safe level threshold.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MPU-6050 IMU Sensor | `mpu6050` | Yes | Yes |
| SSD1306 OLED Display (128×64) | `oled` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 | VCC (+) | 3.3V (3V3) | Red | Module power |
| MPU-6050 | GND (−) | GND | Black | Ground reference |
| MPU-6050 | SDA | GP4 | Orange | Shared I2C Bus 0 data |
| MPU-6050 | SCL | GP5 | Blue | Shared I2C Bus 0 clock |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| SSD1306 OLED | VCC / GND | 3.3V / GND | Red / Black | OLED power |
| Active Buzzer | VCC (+) | GP15 | Red | Level alarm output |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The MPU-6050 (address 0x68) and OLED (address 0x3C) share I2C Bus 0 on GP4/GP5 without conflict because each device has a unique address. Connect the buzzer to GP15. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime
import struct
import math
import ssd1306

i2c  = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

MPU_ADDR = 0x68

# Wake MPU-6050 from sleep
i2c.writeto_mem(MPU_ADDR, 0x6B, b'\x00')

buzzer = Pin(15, Pin.OUT)
buzzer.value(0)

LEVEL_THRESHOLD_DEG = 15.0  # Alarm if tilt exceeds this

def read_accel():
    raw = i2c.readfrom_mem(MPU_ADDR, 0x3B, 6)
    ax  = struct.unpack('>h', raw[0:2])[0]
    ay  = struct.unpack('>h', raw[2:4])[0]
    az  = struct.unpack('>h', raw[4:6])[0]
    return ax, ay, az

def calc_angles(ax, ay, az):
    """Calculate pitch and roll from raw accelerometer values."""
    ax_g  = ax / 16384.0
    ay_g  = ay / 16384.0
    az_g  = az / 16384.0
    pitch = math.degrees(math.atan2(ax_g, math.sqrt(ay_g**2 + az_g**2)))
    roll  = math.degrees(math.atan2(ay_g, math.sqrt(ax_g**2 + az_g**2)))
    return pitch, roll

def draw_bubble(pitch, roll):
    oled.fill(0)

    # Title
    oled.text("INCLINOMETER", 0, 0)
    oled.hline(0, 10, 128, 1)

    # Angle readings
    oled.text("P:{:.1f}d".format(pitch), 0, 14)
    oled.text("R:{:.1f}d".format(roll), 64, 14)

    # Bubble level display — 80x30 box centred at (64, 45)
    CX, CY = 64, 45
    BOX_W, BOX_H = 60, 20

    oled.rect(CX - BOX_W//2, CY - BOX_H//2, BOX_W, BOX_H, 1)

    # Bubble position: clamp to box
    bx = int(CX + max(-BOX_W//2 + 5, min(BOX_W//2 - 5, roll * 1.5)))
    by = int(CY + max(-BOX_H//2 + 4, min(BOX_H//2 - 4, -pitch * 0.8)))

    oled.fill_rect(bx - 4, by - 4, 8, 8, 1)  # Bubble indicator

    # Crosshair centre
    oled.hline(CX - 3, CY, 6, 1)
    oled.vline(CX, CY - 3, 6, 1)

    oled.show()

oled.fill(0); oled.text("Inclinometer", 8, 24); oled.show()
utime.sleep(1)

print("Digital inclinometer active.")

while True:
    ax, ay, az = read_accel()
    pitch, roll = calc_angles(ax, ay, az)

    # Level alarm
    tilted = abs(pitch) > LEVEL_THRESHOLD_DEG or abs(roll) > LEVEL_THRESHOLD_DEG
    buzzer.value(1 if tilted else 0)

    draw_bubble(pitch, roll)

    print("Pitch:{:.1f} Roll:{:.1f} Tilted:{}".format(pitch, roll, tilted))
    utime.sleep_ms(100)
```

## What to Click in MbedO
1. Drag **Pico**, **MPU-6050**, **SSD1306 OLED**, and **Active Buzzer** onto the canvas.
2. Connect MPU-6050 and OLED to **GP4/GP5** (shared I2C bus). Connect Buzzer to **GP15**. Connect power and GND.
3. Paste code, select **MicroPython** mode, click **Run**.
4. Adjust the MPU-6050 Ac_x and Ac_y sliders to simulate tilt. Observe the bubble moving on the OLED and the buzzer activating above 15°.

## Expected Output
```
Digital inclinometer active.
Pitch:0.0 Roll:0.0 Tilted:False
Pitch:12.5 Roll:-8.2 Tilted:False
Pitch:20.1 Roll:5.0 Tilted:True
```
(On OLED: a filled bubble square moves within a rectangular level box. Centre = level.)

## Expected Canvas Behavior
- Adjusting the MPU-6050 sliders moves the graphical bubble on the OLED display. The buzzer sounds when the bubble exceeds the 15° threshold on either axis.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `math.atan2(ax_g, math.sqrt(ay_g**2 + az_g**2))` | Calculates pitch angle from gravity vector components using the arctangent formula. |
| `roll * 1.5` | Scales the roll angle to pixel displacement for the bubble position on screen. |

## Hardware & Safety Concept: Tilt Sensing with Accelerometers
Accelerometers measure the vector sum of linear acceleration and gravitational acceleration. When stationary, the only force acting is gravity (1g). By comparing the X, Y, Z components of this gravity vector to the known 1g magnitude, the orientation can be calculated using trigonometry. This technique works well for static or slow-moving objects but is inaccurate during rapid movement.

## Try This! (Challenges)
1. **Gyroscope Fusion**: Read the MPU-6050 gyroscope registers alongside the accelerometer and implement a complementary filter (angle = 0.98 × (angle + gyro_rate × dt) + 0.02 × accel_angle) for smoother angle estimation.
2. **Calibration Mode**: Add a button that zeros the offset by recording the current pitch/roll as the level reference.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| OLED stays blank | I2C address conflict | Run `i2c.scan()` and verify addresses 0x68 (MPU) and 0x3C (OLED) both appear. |
| Angles always 0 | Sleep bit not cleared | Ensure `i2c.writeto_mem(MPU_ADDR, 0x6B, b'\x00')` is called at startup. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [109 - Pico Ultrasonic Distance OLED](../intermediate/109-pico-ultrasonic-distance-oled.md)
- [126 - Pico MPU-6050 Tilt Security Vault](126-pico-mpu6050-tilt-security-vault.md)
