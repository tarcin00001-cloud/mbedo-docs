# 151 - Pico Servo Pan-Tilt Camera Mount HC-SR04

Build a two-axis camera pan-tilt mount that sweeps a pan servo automatically, measures distance with an HC-SR04 sensor on each heading, maps the distances to a polar OLED radar plot, and holds the servo on the nearest detected object.

## Goal
Learn how to coordinate two PWM servo motors for pan and tilt axes, synchronise ultrasonic distance measurements with servo position to build a polar range map, and render the map as a radar sweep on an SSD1306 OLED in MicroPython.

## What You Will Build
A servo pan-tilt radar mount:
- **Pan Servo (GP10)**: Sweeps 0°–180° and back continuously.
- **Tilt Servo (GP11)**: Set to a fixed tilt angle, adjustable via buttons.
- **HC-SR04 Ultrasonic (GP16, GP17)**: Measures distance at each pan heading.
- **Button Up/Down (GP13, GP14)**: Adjusts tilt angle in 10° steps.
- **SSD1306 OLED (GP4, GP5)**: Displays a polar radar sweep plot.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Servo Motor × 2 (SG90) | `servo` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `hcsr04` | Yes | Yes |
| Tactile Button × 2 | `button` | Yes | Yes |
| SSD1306 OLED Display (128×64) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Pan Servo | Control (PWM) | GP10 | Orange | Pan axis servo signal |
| Pan Servo | VCC | 5V (VBUS) | Red | Servo power |
| Tilt Servo | Control (PWM) | GP11 | Blue | Tilt axis servo signal |
| Tilt Servo | VCC | 5V (VBUS) | Red | Shared servo power rail |
| HC-SR04 | VCC | 5V (VBUS) | Red | Sensor power |
| HC-SR04 | GND | GND | Black | Ground reference |
| HC-SR04 | TRIG | GP16 | Orange | Trigger pulse output |
| HC-SR04 | ECHO | GP17 | Yellow | Echo input (1kΩ/2kΩ divider) |
| Button Up | Terminal 1 / 2 | GP13 / GND | White / Black | Increase tilt angle |
| Button Down | Terminal 1 / 2 | GP14 / GND | White / Black | Decrease tilt angle |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Servos | GND | GND | Black | Shared return path to ground |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Add a 1kΩ/2kΩ voltage divider on HC-SR04 ECHO (5V → 3.3V). Both servos share the 5V power rail. Connect OLED to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, PWM, I2C
import utime, machine, math, ssd1306

pan_servo  = PWM(Pin(10)); pan_servo.freq(50)
tilt_servo = PWM(Pin(11)); tilt_servo.freq(50)

trig = Pin(16, Pin.OUT)
echo = Pin(17, Pin.IN)

btn_up = Pin(13, Pin.IN, Pin.PULL_UP)
btn_dn = Pin(14, Pin.IN, Pin.PULL_UP)

i2c  = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

def set_servo(pwm, angle):
    duty = 1638 + int(angle * (8192 - 1638) / 180)
    pwm.duty_u16(duty)

def measure_dist():
    trig.value(0); utime.sleep_us(2)
    trig.value(1); utime.sleep_us(10)
    trig.value(0)
    d = machine.time_pulse_us(echo, 1, 30000)
    return 400.0 if d < 0 else d / 58.0

tilt_angle = 45
set_servo(tilt_servo, tilt_angle)

# Radar data: distance at each pan angle (0-180, step 5)
ANGLES   = list(range(0, 181, 5))   # 37 points
radar    = [200.0] * len(ANGLES)     # Default: far
MAX_DIST = 200.0                     # Plot scale (cm)

# OLED radar centre
CX, CY = 64, 56
RADIUS  = 50

last_btn = 0

def polar_to_xy(angle_deg, dist_cm):
    r   = min(RADIUS, dist_cm * RADIUS / MAX_DIST)
    rad = math.radians(180 - angle_deg)  # Flip so 0° is left
    x   = int(CX + r * math.cos(rad))
    y   = int(CY - r * math.sin(rad))
    return x, y

def draw_radar(current_idx):
    oled.fill(0)
    # Semicircle arcs
    for arc_r in (RADIUS // 3, 2 * RADIUS // 3, RADIUS):
        for deg in range(0, 181, 3):
            rad = math.radians(deg)
            x   = int(CX + arc_r * math.cos(rad))
            y   = int(CY - arc_r * math.sin(rad))
            oled.pixel(x, y, 1)
    # Centre lines
    oled.hline(CX - RADIUS, CY, RADIUS * 2, 1)
    oled.vline(CX, CY - RADIUS, RADIUS, 1)

    # Plot radar returns
    for i, angle in enumerate(ANGLES):
        x, y = polar_to_xy(angle, radar[i])
        oled.fill_rect(x - 1, y - 1, 3, 3, 1)

    # Sweep line
    sx, sy = polar_to_xy(ANGLES[current_idx], RADIUS * 1.2)
    oled.line(CX, CY, sx, sy, 1)

    # Header
    a   = ANGLES[current_idx]
    d   = radar[current_idx]
    oled.text("{}d {:.0f}cm T:{}".format(a, d, tilt_angle), 0, 0)
    oled.show()

print("Pan-tilt radar active.")
oled.fill(0); oled.text("Pan-Tilt Radar", 4, 28); oled.show(); utime.sleep(1)

direction = 1  # 1 = increasing angle, -1 = decreasing
idx       = 0

while True:
    now = utime.ticks_ms()

    # Tilt adjustment
    if btn_up.value() == 0 and utime.ticks_diff(now, last_btn) > 300:
        tilt_angle = min(135, tilt_angle + 10)
        set_servo(tilt_servo, tilt_angle); last_btn = now
    if btn_dn.value() == 0 and utime.ticks_diff(now, last_btn) > 300:
        tilt_angle = max(20, tilt_angle - 10)
        set_servo(tilt_servo, tilt_angle); last_btn = now

    # Pan to current angle and measure
    set_servo(pan_servo, ANGLES[idx])
    utime.sleep_ms(60)  # Servo settle time
    radar[idx] = measure_dist()

    draw_radar(idx)
    print("Pan:{:3d}° Dist:{:.0f}cm Tilt:{}°".format(ANGLES[idx], radar[idx], tilt_angle))

    # Advance pan
    idx += direction
    if idx >= len(ANGLES) - 1:
        direction = -1
    elif idx <= 0:
        direction = 1
```

## What to Click in MbedO
1. Drag **Pico**, **two Servos**, **HC-SR04**, **two Buttons**, and **SSD1306 OLED** onto the canvas.
2. Connect Pan Servo to **GP10**, Tilt Servo to **GP11**, HC-SR04 TRIG to **GP16** and ECHO to **GP17**, Buttons to **GP13/GP14**, OLED to **GP4/GP5**.
3. Paste code, click **Run**. The pan servo sweeps 0°–180°. Adjust the HC-SR04 slider to simulate distance at different angles.

## Expected Output
```
Pan-tilt radar active.
Pan:  0° Dist:120cm Tilt:45°
Pan:  5° Dist:85cm Tilt:45°
Pan: 90° Dist:200cm Tilt:45°
```
(On OLED: a semicircular radar display with dots plotted at each measured distance and a sweep line.)

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `polar_to_xy(angle_deg, dist_cm)` | Converts polar coordinates (angle, distance) to OLED pixel coordinates for radar plotting. |
| `direction = -1` when `idx >= len(ANGLES) - 1` | Reverses the sweep direction at the 180° limit, producing continuous back-and-forth scanning. |

## Hardware & Safety Concept: Servo Settle Time
After commanding a servo to a new angle, the PWM signal drives the motor but the mechanical rotation takes time. `utime.sleep_ms(60)` gives a small SG90 servo sufficient time to reach the target angle before the ultrasonic measurement is taken, avoiding phantom short-distance readings caused by measuring while the mount is still moving.

## Try This! (Challenges)
1. **Nearest Object Lock**: After a full sweep, automatically drive the pan servo to the angle with the minimum detected distance.
2. **Persistent Radar**: Draw radar dots in white for new measurements and fade old ones by only partially clearing the frame.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servos jitter | Power supply insufficient | Use a dedicated 5V 1A supply for the servos rather than sharing Pico 5V. |
| Radar always shows max distance | HC-SR04 echo not received | Verify voltage divider on ECHO line. Check clear line of sight for sensor. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [117 - Pico Servo Radar Scanner OLED](../intermediate/117-pico-servo-radar-scanner-oled.md)
- [115 - Pico Parking Sensor HC-SR04 Buzzer LCD](../intermediate/115-pico-parking-sensor-hcsr04-buzzer-lcd.md)
