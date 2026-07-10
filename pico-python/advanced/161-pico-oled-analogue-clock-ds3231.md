# 161 - Pico OLED Analogue Clock DS3231

Build a real-time analogue clock that reads hours, minutes, and seconds from a DS3231 RTC and renders a graphical clock face — with hour, minute, and second hands, tick marks, and a digital time readout — on an SSD1306 OLED.

## Goal
Learn how to calculate analogue clock hand endpoints using trigonometry, draw a complete clock face with tick marks and numerals on an SSD1306 OLED, and synchronise the animation to a DS3231 RTC over I2C in MicroPython.

## What You Will Build
A graphical analogue clock display:
- **DS3231 RTC (GP4, GP5 I2C)**: Provides accurate time via I2C.
- **SSD1306 OLED (GP4, GP5 shared bus)**: Renders the analogue clock face.
- **Button (GP13)**: Short press = show/hide seconds hand; long press = enter time-set mode.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DS3231 RTC Module | `ds3231` | Yes | Yes |
| SSD1306 OLED Display (128×64) | `oled` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS3231 | VCC / GND | 3.3V / GND | Red / Black | Module power |
| DS3231 | SDA / SCL | GP4 / GP5 | Orange / Blue | Shared I2C Bus 0 |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| SSD1306 OLED | VCC / GND | 3.3V / GND | Red / Black | OLED power |
| Button | Terminal 1 / 2 | GP13 / GND | White / Black | Mode control |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** DS3231 and OLED share I2C Bus 0 on GP4/GP5. DS3231 address is 0x68; OLED is 0x3C. Button on GP13 with PULL_UP. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime, math, ssd1306

i2c  = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)
btn  = Pin(13, Pin.IN, Pin.PULL_UP)

DS3231_ADDR = 0x68
CX, CY = 32, 32   # Clock face centre (left half of OLED)
RADIUS  = 28

def bcd_to_dec(b): return (b >> 4) * 10 + (b & 0x0F)
def dec_to_bcd(d): return ((d // 10) << 4) | (d % 10)

def rtc_read():
    d  = i2c.readfrom_mem(DS3231_ADDR, 0x00, 7)
    return bcd_to_dec(d[2] & 0x3F), bcd_to_dec(d[1]), bcd_to_dec(d[0] & 0x7F)

def rtc_set(h, m, s):
    i2c.writeto_mem(DS3231_ADDR, 0x00, bytes([
        dec_to_bcd(s), dec_to_bcd(m), dec_to_bcd(h), 1, 1, 1, 26]))

def hand_end(cx, cy, length, angle_deg):
    """Angle 0 = 12 o'clock, increases clockwise."""
    rad = math.radians(angle_deg - 90)
    return int(cx + length * math.cos(rad)), int(cy + length * math.sin(rad))

def draw_clock(hr, mn, sec, show_sec):
    oled.fill(0)

    # --- Clock face (left side) ---
    # Outer ring
    for deg in range(0, 360, 6):
        x, y = hand_end(CX, CY, RADIUS, deg)
        oled.pixel(x, y, 1)

    # Hour tick marks (every 30°)
    for i in range(12):
        a    = i * 30
        ox, oy = hand_end(CX, CY, RADIUS - 1, a)
        ix, iy = hand_end(CX, CY, RADIUS - 4, a)
        oled.line(ox, oy, ix, iy, 1)

    # 12 / 3 / 6 / 9 labels
    for label, a in [("12", 0), ("3", 90), ("6", 180), ("9", 270)]:
        tx, ty = hand_end(CX, CY, RADIUS - 9, a)
        oled.text(label, tx - len(label) * 3, ty - 4)

    # Hands
    hr_angle  = (hr % 12) * 30 + mn * 0.5
    mn_angle  = mn * 6 + sec * 0.1
    sec_angle = sec * 6

    # Hour hand (short)
    hx, hy = hand_end(CX, CY, RADIUS - 10, hr_angle)
    oled.line(CX, CY, hx, hy, 1)

    # Minute hand (long)
    mx, my = hand_end(CX, CY, RADIUS - 4, mn_angle)
    oled.line(CX, CY, mx, my, 1)

    # Second hand (optional — thin)
    if show_sec:
        sx, sy = hand_end(CX, CY, RADIUS - 2, sec_angle)
        oled.line(CX, CY, sx, sy, 1)

    # Centre hub
    oled.fill_rect(CX - 2, CY - 2, 4, 4, 1)

    # --- Digital readout (right side) ---
    oled.vline(68, 0, 64, 1)  # Divider
    oled.text("{:02d}:{:02d}".format(hr, mn), 72, 10)
    oled.text(":{:02d}".format(sec), 90, 22)

    # Day and uptime
    uptime_s = utime.ticks_ms() // 1000
    oled.text("UP:{:5d}s".format(uptime_s % 100000), 70, 40)
    oled.text("RTC:DS3231", 70, 52)

    oled.show()

# Auto-set RTC on first run if uninitialised
h, m, s = rtc_read()
if h == 0 and m == 0 and s < 2:
    rtc_set(8, 0, 0)

show_sec  = True
last_btn  = 0
btn_press = 0

oled.fill(0); oled.text("OLED Clock", 24, 28); oled.show(); utime.sleep(1)
print("Analogue OLED clock active.")

while True:
    now    = utime.ticks_ms()
    hr, mn, sec = rtc_read()

    # Button press handling
    if btn.value() == 0:
        if btn_press == 0:
            btn_press = now
        elif utime.ticks_diff(now, btn_press) > 1500:
            # Long press: increment hours (time-set mode)
            rtc_set((hr + 1) % 24, mn, 0)
            btn_press = 0
            utime.sleep_ms(300)
    else:
        if 0 < utime.ticks_diff(now, btn_press) < 1500 and btn_press > 0:
            show_sec  = not show_sec  # Short press: toggle seconds hand
        btn_press = 0

    draw_clock(hr, mn, sec, show_sec)
    utime.sleep_ms(200)
```

## What to Click in MbedO
1. Drag **Pico**, **DS3231 RTC**, **SSD1306 OLED**, and **Push Button** onto the canvas.
2. Connect DS3231 and OLED to **GP4/GP5** (shared I2C). Connect Button to **GP13**. Connect power and GND.
3. Paste code, click **Run**. An analogue clock face appears on the left half of the OLED. Short-press the button to toggle the seconds hand. Long-press to advance the hour.

## Expected Output
```
Analogue OLED clock active.
```
(On OLED: an analogue clock face with hour, minute, and second hands. Digital time on the right half.)

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `hr_angle = (hr % 12) * 30 + mn * 0.5` | Hour hand angle: each hour = 30°; each minute adds 0.5° so the hand moves smoothly between hour marks. |
| `hand_end(cx, cy, length, angle_deg)` | Converts polar clock hand coordinates to OLED Cartesian pixels with 12 o'clock at top (−90° offset). |

## Hardware & Safety Concept: OLED Burn-In Prevention
SSD1306 OLEDs can develop burn-in on static elements after thousands of hours. To mitigate this, periodically shift the display content by 1 pixel, reduce brightness for non-interactive periods, or add a screensaver mode activated after 60 seconds of no button presses.

## Try This! (Challenges)
1. **Alarm Hand**: Add a triangle marker at the alarm time on the clock face using `oled.triangle()` to indicate when an alarm is set.
2. **Stopwatch Mode**: Long-press to enter a stopwatch sub-mode that counts elapsed time using `utime.ticks_ms()` while the RTC continues in the background.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Clock runs backwards | `angle - 90` sign error | Ensure `math.radians(angle_deg - 90)` in `hand_end()`. Do not negate. |
| Hands overlap at 12 | Hour offset off by 30° | Check `(hr % 12) * 30` — `hr = 12` must map to `0°` via the modulo. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [160 - Pico DS3231 RTC Dual Alarm LCD](160-pico-ds3231-rtc-dual-alarm-lcd.md)
- [139 - Pico MPU-6050 Digital Inclinometer OLED](139-pico-mpu6050-digital-inclinometer-oled.md)
