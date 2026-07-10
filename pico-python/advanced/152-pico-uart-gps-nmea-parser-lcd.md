# 152 - Pico UART GPS NMEA Parser LCD

Build a GPS receiver display that reads NMEA sentences from a serial GPS module (e.g. NEO-6M), parses the GPRMC and GPGGA sentences, extracts latitude, longitude, speed, altitude, and satellite count, and displays the data on an I2C LCD.

## Goal
Learn how to receive and parse NMEA 0183 GPS sentences over UART, extract decimal coordinates from the DDMM.MMMM format, handle optional sentence fields, and display live GPS data on an I2C character LCD in MicroPython.

## What You Will Build
A GPS display station:
- **NEO-6M GPS Module (GP0 TX, GP1 RX)**: Sends NMEA 0183 sentences at 9600 baud.
- **I2C 16x2 LCD (GP4, GP5)**: Pages through latitude/longitude, speed, altitude, and satellite count.
- **Button (GP13)**: Cycles display pages.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| NEO-6M GPS Module | `uart` | Yes (UART simulation) | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NEO-6M GPS | TXD | GP1 (UART0 RX) | Yellow | GPS data to Pico |
| NEO-6M GPS | RXD | GP0 (UART0 TX) | Orange | Pico to GPS (optional) |
| NEO-6M GPS | VCC | 3.3V (3V3) | Red | GPS module power |
| NEO-6M GPS | GND | GND | Black | Ground reference |
| Button | Terminal 1 / 2 | GP13 / GND | White / Black | Page cycle button |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The NEO-6M outputs 3.3V UART by default — no level shifter needed. Connect TXD → GP1 (Pico UART0 RX). Button on GP13 with PULL_UP. LCD on GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, UART, I2C
import utime
from machine_lcd import I2cLcd

gps = UART(0, baudrate=9600, tx=Pin(0), rx=Pin(1))
btn = Pin(13, Pin.IN, Pin.PULL_UP)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Parsed GPS data cache
gps_data = {
    'lat'   : 0.0, 'lon'  : 0.0,
    'speed' : 0.0, 'alt'  : 0.0,
    'sats'  : 0,   'valid': False,
    'time'  : '--:--:--', 'date': '------'
}

PAGE_COUNT = 3
page       = 0
last_btn   = 0

def nmea_checksum_ok(sentence):
    """Validate NMEA sentence checksum."""
    try:
        data, chk = sentence[1:].rsplit('*', 1)
        calc = 0
        for c in data:
            calc ^= ord(c)
        return calc == int(chk.strip(), 16)
    except Exception:
        return False

def ddmm_to_decimal(ddmm, direction):
    """Convert NMEA DDMM.MMMM to decimal degrees."""
    try:
        dot = ddmm.index('.')
        deg = float(ddmm[:dot - 2])
        mins = float(ddmm[dot - 2:])
        dec = deg + mins / 60.0
        if direction in ('S', 'W'):
            dec = -dec
        return dec
    except Exception:
        return 0.0

def parse_gprmc(fields):
    """Parse $GPRMC sentence."""
    if len(fields) < 9:
        return
    gps_data['time']  = fields[1][:6] if len(fields[1]) >= 6 else '--:--:--'
    gps_data['valid'] = (fields[2] == 'A')
    if gps_data['valid']:
        gps_data['lat']   = ddmm_to_decimal(fields[3], fields[4])
        gps_data['lon']   = ddmm_to_decimal(fields[5], fields[6])
        gps_data['speed'] = float(fields[7]) * 1.852 if fields[7] else 0.0  # knots → km/h
        gps_data['date']  = fields[9] if len(fields) > 9 else '------'

def parse_gpgga(fields):
    """Parse $GPGGA sentence."""
    if len(fields) < 10:
        return
    try:
        gps_data['alt']  = float(fields[9]) if fields[9] else 0.0
        gps_data['sats'] = int(fields[7])   if fields[7] else 0
    except (ValueError, IndexError):
        pass

def process_line(line):
    line = line.strip()
    if not line.startswith('$'):
        return
    if not nmea_checksum_ok(line):
        return
    fields = line.split(',')
    sentence = fields[0][1:]
    if sentence == 'GPRMC':
        parse_gprmc(fields)
    elif sentence == 'GPGGA':
        parse_gpgga(fields)

buf = b''

def update_lcd():
    lcd.clear()
    if not gps_data['valid']:
        lcd.putstr("GPS: No Fix     ")
        lcd.move_to(0, 1)
        lcd.putstr("Sats: {:2d}  Wait..".format(gps_data['sats']))
        return
    if page == 0:
        lcd.putstr("Lat:{:.4f}".format(gps_data['lat']))
        lcd.move_to(0, 1)
        lcd.putstr("Lon:{:.4f}".format(gps_data['lon']))
    elif page == 1:
        lcd.putstr("Spd:{:.1f}km/h".format(gps_data['speed']))
        lcd.move_to(0, 1)
        lcd.putstr("Alt:{:.0f}m Sat:{:2d}".format(gps_data['alt'], gps_data['sats']))
    elif page == 2:
        t = gps_data['time']
        d = gps_data['date']
        lcd.putstr("Time:{}-{}-{}".format(t[0:2], t[2:4], t[4:6]))
        lcd.move_to(0, 1)
        lcd.putstr("Date:{}-{}-{}".format(d[0:2], d[2:4], d[4:6]))

lcd.clear(); lcd.putstr("GPS Parser"); lcd.move_to(0, 1); lcd.putstr("Waiting for fix")
utime.sleep(1)
print("GPS NMEA parser active.")

while True:
    now = utime.ticks_ms()

    # Page button
    if btn.value() == 0 and utime.ticks_diff(now, last_btn) > 400:
        page     = (page + 1) % PAGE_COUNT
        last_btn = now

    # Read GPS UART
    if gps.any():
        chunk = gps.read(64)
        if chunk:
            buf += chunk
            while b'\n' in buf:
                line, buf = buf.split(b'\n', 1)
                try:
                    process_line(line.decode('ascii', 'ignore'))
                except Exception:
                    pass

    update_lcd()
    utime.sleep_ms(200)
```

## What to Click in MbedO
1. Drag **Pico**, **UART GPS** component, **Push Button**, and **I2C LCD** onto the canvas.
2. Connect GPS TXD to **GP1**, Button to **GP13**, LCD to **GP4/GP5**. Connect power and GND.
3. Paste code, click **Run**. In MbedO, inject NMEA sentences through the UART simulation input.
4. Press Button to cycle through Lat/Lon, Speed/Alt, and Time/Date pages.

## Expected Output
```
GPS NMEA parser active.
```
(On LCD Page 0: `Lat:51.5074` / `Lon:-0.1278`, Page 1: `Spd:0.0km/h` / `Alt:15m Sat:8`, Page 2: `Time:12-34-56` / `Date:10-07-26`)

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `ddmm_to_decimal(ddmm, direction)` | Converts NMEA's DDMM.MMMM format (degrees + minutes) to standard decimal degrees used in mapping. |
| `buf.split(b'\n', 1)` | Extracts complete NMEA lines from the accumulating byte buffer, preserving any partial line for the next iteration. |

## Hardware & Safety Concept: NMEA 0183 Protocol
NMEA 0183 is the universal serial standard for GPS devices. Each sentence begins with `$` and ends with `*XX` (two hex checksum digits). The checksum is an XOR of all bytes between `$` and `*`. Always validate the checksum before parsing to avoid corrupted position data.

## Try This! (Challenges)
1. **Waypoint Distance**: Store a home waypoint on first fix and calculate the Haversine great-circle distance to display the distance from home.
2. **Track Logger**: Write latitude and longitude to EEPROM every 30 seconds to build a travel track log.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD shows "No Fix" indefinitely | GPS module needs cold start time | Allow 30–90 seconds outdoors with clear sky for first fix on a cold start. |
| Garbled characters on UART | Baud rate mismatch | Default NEO-6M baud is 9600. Verify UART initialization matches. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [128 - Pico Bluetooth Home Controller HC-05 Relay LCD](128-pico-bluetooth-home-controller-hc05-relay-lcd.md)
- [132 - Pico BT Environment Monitor HC-05 DHT22 MQ-2 LCD](132-pico-bt-environment-monitor-hc05-dht22-mq2-lcd.md)
