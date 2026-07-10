# 140 - Pico Full Weather Console DHT22 BMP180 LDR OLED

Build a comprehensive five-parameter weather console that fuses temperature, humidity, barometric pressure, altitude, and light level readings into a paginated dashboard displayed on an SSD1306 OLED with a status bar.

## Goal
Learn how to fuse data from three independent sensors (DHT22, BMP180, and LDR) into a multi-page OLED dashboard, implement page cycling with a button, calculate derived values (altitude, heat index, UV estimate), and display a status bar in MicroPython.

## What You Will Build
A five-sensor weather console:
- **DHT22 (GP16)**: Temperature (°C) and humidity (%).
- **BMP180 (GP4, GP5 I2C)**: Barometric pressure (hPa) and calculated altitude (m).
- **LDR (GP26 ADC)**: Ambient light level (%).
- **Button (GP14)**: Cycles between display pages.
- **SSD1306 OLED (GP4, GP5 shared I2C)**: Multi-page dashboard.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| BMP180 Sensor | `bmp180` | Yes | Yes |
| LDR Photoresistor | `ldr` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| SSD1306 OLED Display (128×64) | `oled` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | Optional in MbedO | Yes (LDR pull-down) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 | VCC (+) | 3.3V (3V3) | Red | Sensor power |
| DHT22 | GND (−) | GND | Black | Ground reference |
| DHT22 | DATA | GP16 | Yellow | 1-Wire data signal |
| BMP180 | VCC (+) | 3.3V (3V3) | Red | Sensor power |
| BMP180 | GND (−) | GND | Black | Ground reference |
| BMP180 | SDA | GP4 | Orange | Shared I2C Bus 0 data |
| BMP180 | SCL | GP5 | Blue | Shared I2C Bus 0 clock |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| SSD1306 OLED | VCC / GND | 3.3V / GND | Red / Black | OLED power |
| LDR | Pin 1 | 3.3V (3V3) | Red | Power line |
| LDR | Pin 2 | GP26 | Yellow | ADC input with 10kΩ to GND |
| 10 kΩ Resistor | Either leg | GP26 to GND | — | Pull-down for LDR divider |
| Button | Terminal 1 | GP14 | White | Page cycle button |
| Button | Terminal 2 | GND | Black | Button return |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** BMP180 and OLED share I2C Bus 0 on GP4/GP5. DHT22 uses a separate 1-Wire protocol on GP16. LDR forms a voltage divider with a 10 kΩ pull-down to GP26. Button uses GP14 with internal pull-up. All grounds are shared.

## Code
```python
from machine import Pin, ADC, I2C
import utime, dht, math, ssd1306, struct
from bmp085 import BMP180

# Sensors
dht_sensor = dht.DHT22(Pin(16))
ldr_adc    = ADC(26)
btn        = Pin(14, Pin.IN, Pin.PULL_UP)

# Shared I2C Bus 0
i2c  = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)
bmp  = BMP180(i2c)
bmp.oversample_setting = 2

SEA_LEVEL_PA = 101325.0

def calc_altitude(pa):
    return 44330.0 * (1.0 - (pa / SEA_LEVEL_PA) ** (1.0 / 5.255))

def calc_heat_index(t, h):
    """Simplified Steadman heat index approximation."""
    return t + 0.33 * (h / 100 * 6.105 * math.exp(17.27 * t / (237.7 + t))) - 4.00

# Sensor cache
temp_c, humid, pressure_hpa, altitude_m, light_pct, heat_idx = 0.0, 0.0, 0.0, 0.0, 0, 0.0

def refresh_sensors():
    global temp_c, humid, pressure_hpa, altitude_m, light_pct, heat_idx
    try:
        dht_sensor.measure()
        temp_c  = dht_sensor.temperature()
        humid   = dht_sensor.humidity()
        heat_idx = calc_heat_index(temp_c, humid)
    except OSError:
        pass
    pressure_hpa = bmp.pressure / 100.0
    altitude_m   = calc_altitude(bmp.pressure)
    light_pct    = min(100, int(ldr_adc.read_u16() * 100 / 65535))

# Page definitions
PAGE_COUNT = 3
page       = 0

def draw_page_0():
    oled.fill(0)
    oled.text("WEATHER CONSOLE", 0, 0)
    oled.hline(0, 10, 128, 1)
    oled.text("T: {:.1f} C".format(temp_c), 0, 14)
    oled.text("H: {:.0f} %".format(humid), 0, 26)
    oled.text("HI:{:.1f} C".format(heat_idx), 0, 38)
    oled.text("Pg 1/3  BTN=Next", 0, 54)
    oled.show()

def draw_page_1():
    oled.fill(0)
    oled.text("BAROMETRIC DATA", 0, 0)
    oled.hline(0, 10, 128, 1)
    oled.text("P: {:.1f} hPa".format(pressure_hpa), 0, 14)
    oled.text("A: {:.0f} m".format(altitude_m), 0, 26)
    bmp_t = bmp.temperature
    oled.text("BT:{:.1f} C".format(bmp_t), 0, 38)
    oled.text("Pg 2/3  BTN=Next", 0, 54)
    oled.show()

def draw_page_2():
    oled.fill(0)
    oled.text("ENVIRONMENT", 0, 0)
    oled.hline(0, 10, 128, 1)
    oled.text("Light: {:3d} %".format(light_pct), 0, 14)
    bar = min(112, int(light_pct * 112 / 100))
    oled.rect(8, 28, 112, 10, 1)
    oled.fill_rect(8, 28, bar, 10, 1)
    # Trend summary
    trend = "BRIGHT" if light_pct > 60 else "CLOUDY" if light_pct > 25 else "DARK  "
    oled.text("Cond: " + trend, 0, 42)
    oled.text("Pg 3/3  BTN=Next", 0, 54)
    oled.show()

PAGES = [draw_page_0, draw_page_1, draw_page_2]

# Timing
REFRESH_MS   = 2000
last_refresh = utime.ticks_ms() - REFRESH_MS
last_btn     = 0

oled.fill(0); oled.text("Weather Console", 0, 0); oled.text("Loading...", 0, 24); oled.show()
utime.sleep(2)

refresh_sensors()
print("Full weather console active.")

while True:
    now = utime.ticks_ms()

    # Sensor refresh every 2 seconds
    if utime.ticks_diff(now, last_refresh) >= REFRESH_MS:
        refresh_sensors()
        last_refresh = now

    # Button: cycle page
    if btn.value() == 0 and utime.ticks_diff(now, last_btn) > 400:
        page     = (page + 1) % PAGE_COUNT
        last_btn = now
        print("Page:", page + 1)

    # Draw current page
    PAGES[page]()

    print("T:{:.1f}C H:{:.0f}% P:{:.0f}hPa Alt:{:.0f}m Light:{}%".format(
        temp_c, humid, pressure_hpa, altitude_m, light_pct))

    utime.sleep_ms(150)
```

## What to Click in MbedO
1. Drag **Pico**, **DHT22**, **BMP180**, **LDR Potentiometer**, **Push Button**, and **SSD1306 OLED** onto the canvas.
2. Connect BMP180 and OLED to **GP4/GP5** (shared I2C). Connect DHT22 to **GP16**, LDR to **GP26**, Button to **GP14**. Connect power and GND.
3. Paste code, select **MicroPython** mode, click **Run**.
4. Click the Button to cycle through the three dashboard pages. Adjust sensor sliders to change readings.

## Expected Output
```
Full weather console active.
T:24.5C H:58% P:1013.0hPa Alt:0m Light:45%
```
(On OLED: three alternating pages — Temperature/Humidity/HeatIndex, Barometric/Altitude, Light/Condition.)

## Expected Canvas Behavior
- Clicking the Button component cycles through three OLED pages. Each page displays a different set of climate readings. Adjusting DHT22, BMP180, or LDR sliders updates the corresponding page values.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `PAGES = [draw_page_0, draw_page_1, draw_page_2]` | Stores the page-drawing functions in a list, allowing the active page to be called with `PAGES[page]()`. |
| `utime.ticks_diff(now, last_btn) > 400` | Button debounce: ignores additional presses within 400 ms of the first. |

## Hardware & Safety Concept: Paginated OLED Displays
A 128×64 OLED has limited screen real estate for multi-sensor dashboards. Paginated displays (multiple screens accessible via button press) are the standard approach in weather stations and industrial HMIs. Each page focuses on one data category, avoiding the visual clutter of cramming all readings onto one screen.

## Try This! (Challenges)
1. **Auto-Page Cycling**: Remove the button requirement and auto-advance pages every 5 seconds for a display-only kiosk mode.
2. **Comfort Assessment**: Add a fourth page showing a "Comfort Score" calculated from temperature and humidity using the Feels-Like formula.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| BMP180 not found | I2C address conflict | Run `i2c.scan()` and verify addresses 0x77 (BMP180) and 0x3C (OLED) both appear. |
| DHT22 read fails intermittently | Sensor being polled too fast | The 2-second `REFRESH_MS` satisfies DHT22's minimum sample interval. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [110 - Pico DHT22 BMP180 Weather Station OLED](../intermediate/110-pico-dht22-bmp180-weather-station-oled.md)
- [139 - Pico MPU-6050 Digital Inclinometer OLED](139-pico-mpu6050-digital-inclinometer-oled.md)
