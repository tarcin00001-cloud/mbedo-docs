# 155 - Pico Environmental Data Station SD Card Logger

Build a comprehensive environmental data station that logs temperature, humidity, pressure, and light level readings to a microSD card in CSV format, with a real-time OLED dashboard and configurable logging interval.

## Goal
Learn how to interface a microSD card module over SPI, write and append CSV data files using MicroPython's built-in `os` and file I/O, implement a configurable logging interval, and display a live dashboard on an SSD1306 OLED in MicroPython.

## What You Will Build
A multi-sensor SD card data logger:
- **DHT22 Sensor (GP14)**: Temperature (°C) and humidity (%).
- **BMP180 (GP4, GP5 I2C)**: Barometric pressure (hPa).
- **LDR (GP26)**: Ambient light level (%).
- **MicroSD Card Module (SPI: GP10-GP13)**: CSV data file storage.
- **SSD1306 OLED (GP4, GP5 shared I2C)**: Live multi-sensor dashboard.
- **Button (GP15)**: Forces an immediate log write.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| BMP180 Sensor | `bmp180` | Yes | Yes |
| LDR Photoresistor | `ldr` | Yes | Yes |
| MicroSD Card Module | `sdcard` | Yes | Yes |
| SSD1306 OLED Display (128×64) | `oled` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | Optional in MbedO | Yes (LDR pull-down) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 | VCC / GND / DATA | 3.3V / GND / GP14 | Red/Black/Yellow | 1-Wire sensor |
| BMP180 | VCC / GND | 3.3V / GND | Red / Black | I2C module power |
| BMP180 | SDA / SCL | GP4 / GP5 | Orange / Blue | Shared I2C Bus 0 |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| SSD1306 OLED | VCC / GND | 3.3V / GND | Red / Black | OLED power |
| LDR | Pin 1 | 3.3V (3V3) | Red | Divider top |
| LDR | Pin 2 | GP26 | Yellow | ADC point |
| 10 kΩ Resistor | Either leg | GP26 to GND | — | Divider bottom |
| SD Card Module | SCK / MOSI / MISO | GP10 / GP11 / GP12 | Blue/Orange/Yellow | SPI Bus 1 |
| SD Card Module | CS (SS) | GP13 | White | SPI Chip Select |
| SD Card Module | VCC / GND | 3.3V / GND | Red / Black | Module power |
| Button | Terminal 1 / 2 | GP15 / GND | White / Black | Force log write |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** SD card uses SPI1 (GP10-GP13). BMP180 and OLED share I2C Bus 0 (GP4/GP5). DHT22 on GP14. LDR on GP26. Button on GP15. All grounds are shared.

## Code
```python
from machine import Pin, ADC, I2C, SPI
import utime, dht, ssd1306, os, sdcard
from bmp085 import BMP180

# Sensors
dht_s = dht.DHT22(Pin(14))
ldr   = ADC(26)

# I2C Bus 0: OLED + BMP180
i2c  = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)
bmp  = BMP180(i2c)
bmp.oversample_setting = 2

# SPI Bus 1: SD Card
spi = SPI(1, sck=Pin(10), mosi=Pin(11), miso=Pin(12))
sd  = sdcard.SDCard(spi, Pin(13))
os.mount(sd, '/sd')

btn = Pin(15, Pin.IN, Pin.PULL_UP)

LOG_FILE     = '/sd/envlog.csv'
LOG_INTERVAL = 30   # Seconds between auto-log entries
last_log     = utime.ticks_ms() - LOG_INTERVAL * 1000
log_count    = 0
last_btn     = 0

# Cached readings
temp_c, humid, pressure_hpa, light_pct = 0.0, 0.0, 0.0, 0

def refresh():
    global temp_c, humid, pressure_hpa, light_pct
    try:
        dht_s.measure(); utime.sleep_ms(100)
        temp_c = dht_s.temperature()
        humid  = dht_s.humidity()
    except OSError:
        pass
    pressure_hpa = bmp.pressure / 100.0
    light_pct    = min(100, int(ldr.read_u16() * 100 / 65535))

def write_header():
    try:
        with open(LOG_FILE, 'r') as f:
            return  # File exists, no header needed
    except OSError:
        pass
    with open(LOG_FILE, 'w') as f:
        f.write("timestamp_ms,temp_c,humid_pct,pressure_hpa,light_pct\n")
    print("CSV header written.")

def log_entry():
    global log_count
    ts = utime.ticks_ms()
    with open(LOG_FILE, 'a') as f:
        f.write("{},{:.2f},{:.1f},{:.1f},{}\n".format(
            ts, temp_c, humid, pressure_hpa, light_pct))
    log_count += 1
    print("Logged #{}: T:{:.1f} H:{:.0f} P:{:.0f} L:{}".format(
        log_count, temp_c, humid, pressure_hpa, light_pct))

def draw_oled():
    oled.fill(0)
    oled.text("ENV DATA LOGGER", 0, 0)
    oled.hline(0, 10, 128, 1)
    oled.text("T:{:.1f}C H:{:.0f}%".format(temp_c, humid), 0, 14)
    oled.text("P:{:.0f}hPa".format(pressure_hpa), 0, 26)
    oled.text("Light:{:3d}%".format(light_pct), 0, 38)
    next_log_s = max(0, LOG_INTERVAL - utime.ticks_diff(utime.ticks_ms(), last_log) // 1000)
    oled.text("Log#{} in:{:2d}s".format(log_count, next_log_s), 0, 52)
    oled.show()

write_header()
oled.fill(0); oled.text("SD Logger Init", 4, 28); oled.show()
utime.sleep(2)
print("Environmental data station active. Logging to:", LOG_FILE)

REFRESH_MS   = 2000
last_refresh = utime.ticks_ms() - REFRESH_MS

while True:
    now = utime.ticks_ms()

    if utime.ticks_diff(now, last_refresh) >= REFRESH_MS:
        refresh(); last_refresh = now

    # Auto-log interval
    if utime.ticks_diff(now, last_log) >= LOG_INTERVAL * 1000:
        log_entry(); last_log = now

    # Manual log button
    if btn.value() == 0 and utime.ticks_diff(now, last_btn) > 500:
        log_entry(); last_btn = now

    draw_oled()
    utime.sleep_ms(200)
```

## What to Click in MbedO
1. Drag **Pico**, **DHT22**, **BMP180**, **LDR Potentiometer**, **MicroSD Module**, **SSD1306 OLED**, and **Button** onto the canvas.
2. Connect DHT22 to **GP14**, BMP180 + OLED to **GP4/GP5**, LDR to **GP26**, SD Card to **GP10-GP13**, Button to **GP15**. Connect power and GND.
3. Paste code, click **Run**. Sensor readings appear on OLED. Log entries write to `/sd/envlog.csv` every 30 seconds.

## Expected Output
```
CSV header written.
Environmental data station active. Logging to: /sd/envlog.csv
Logged #1: T:24.5 H:58 P:1013 L:45
Logged #2: T:24.6 H:57 P:1013 L:44
```
(CSV file on SD card: `timestamp_ms,temp_c,humid_pct,pressure_hpa,light_pct` followed by data rows.)

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `os.mount(sd, '/sd')` | Mounts the FAT32 SD card filesystem at `/sd`, enabling standard Python file I/O. |
| `open(LOG_FILE, 'a')` | Opens the log file in append mode, adding new rows without overwriting existing data. |
| `LOG_INTERVAL * 1000` vs `utime.ticks_diff` | Non-blocking interval check in milliseconds, allowing the OLED to update continuously between log writes. |

## Hardware & Safety Concept: SD Card Power-Safe Writes
MicroSD cards write to internal flash in 512-byte sectors. Always close files (`with open(...)` context manager handles this automatically) and avoid power loss during writes to prevent filesystem corruption. For mission-critical logging, flush writes frequently and consider a write-safe log format (append-only, no seeks).

## Try This! (Challenges)
1. **File Rotation**: Create a new log file each day based on a timestamp counter, keeping individual files under 1000 rows.
2. **USB Serial Export**: On button hold (> 3 seconds), read and print the entire CSV file to the USB serial port for easy export without removing the SD card.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| `OSError: [Errno 19]` on mount | SD card not inserted or unsupported format | Use a FAT32-formatted card ≤ 32 GB. Re-insert the card firmly. |
| OLED and BMP180 conflict | I2C address collision | Run `i2c.scan()` to verify addresses — OLED is 0x3C, BMP180 is 0x77. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [140 - Pico Full Weather Console DHT22 BMP180 LDR OLED](140-pico-full-weather-console-dht22-bmp180-ldr-oled.md)
- [141 - Pico I2C EEPROM Data Logger DHT22 LCD](141-pico-i2c-eeprom-data-logger-dht22-lcd.md)
