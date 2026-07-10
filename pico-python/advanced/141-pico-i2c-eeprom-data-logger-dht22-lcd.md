# 141 - Pico I2C EEPROM Data Logger DHT22 LCD

Build a non-volatile sensor data logger that periodically reads a DHT22 sensor, writes timestamped temperature and humidity readings to an AT24C32 I2C EEPROM, and allows the user to replay stored readings on demand via an I2C LCD.

## Goal
Learn how to write and read multi-byte data to an AT24C32 EEPROM using I2C page writes, implement a circular buffer of sensor records in EEPROM, and display stored records on an I2C character LCD in MicroPython.

## What You Will Build
A persistent sensor data logger:
- **DHT22 Sensor (GP16)**: Records temperature and humidity every 10 seconds.
- **AT24C32 EEPROM (GP4, GP5 I2C)**: Stores up to 40 sensor records (non-volatile).
- **Button A (GP13)**: Steps through stored records for review.
- **Button B (GP14)**: Clears the EEPROM log.
- **I2C 16x2 LCD (GP4, GP5 shared bus)**: Displays current reading or replayed records.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| AT24C32 I2C EEPROM | `eeprom` | Yes | Yes |
| Tactile Push Button × 2 | `button` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 | VCC (+) | 3.3V (3V3) | Red | Sensor power |
| DHT22 | GND (−) | GND | Black | Ground reference |
| DHT22 | DATA | GP16 | Yellow | 1-Wire data signal |
| AT24C32 EEPROM | VCC | 3.3V (3V3) | Red | Module power |
| AT24C32 EEPROM | GND | GND | Black | Ground reference |
| AT24C32 EEPROM | SDA | GP4 | Orange | Shared I2C Bus 0 data |
| AT24C32 EEPROM | SCL | GP5 | Blue | Shared I2C Bus 0 clock |
| AT24C32 EEPROM | A0/A1/A2 | GND | Black | Address pins → 0x50 |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| Button A (Review) | Terminal 1 / 2 | GP13 / GND | White / Black | Step through records |
| Button B (Clear) | Terminal 1 / 2 | GP14 / GND | White / Black | Erase EEPROM log |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The AT24C32 EEPROM and LCD share I2C Bus 0 on GP4/GP5. EEPROM address is 0x50 (A0/A1/A2 all tied to GND). The LCD address is 0x27. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime, dht, struct
from machine_lcd import I2cLcd

# Shared I2C Bus 0
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

dht_sensor = dht.DHT22(Pin(16))
btn_review = Pin(13, Pin.IN, Pin.PULL_UP)
btn_clear  = Pin(14, Pin.IN, Pin.PULL_UP)

EEPROM_ADDR  = 0x50
RECORD_BYTES = 8   # 4 bytes temp (float) + 4 bytes humid (float)
MAX_RECORDS  = 40
COUNT_ADDR   = 0   # EEPROM byte 0 stores record count
DATA_OFFSET  = 4   # Records start at byte 4

def eeprom_write(addr, data):
    """Write bytes to EEPROM at given address."""
    i2c.writeto(EEPROM_ADDR, bytes([addr >> 8, addr & 0xFF]) + data)
    utime.sleep_ms(10)  # EEPROM write cycle time

def eeprom_read(addr, nbytes):
    """Read nbytes from EEPROM at given address."""
    i2c.writeto(EEPROM_ADDR, bytes([addr >> 8, addr & 0xFF]))
    return i2c.readfrom(EEPROM_ADDR, nbytes)

def get_count():
    data = eeprom_read(COUNT_ADDR, 4)
    return struct.unpack('>I', data)[0]

def set_count(n):
    eeprom_write(COUNT_ADDR, struct.pack('>I', n))

def write_record(idx, temp, humid):
    addr = DATA_OFFSET + idx * RECORD_BYTES
    eeprom_write(addr, struct.pack('>ff', temp, humid))

def read_record(idx):
    addr = DATA_OFFSET + idx * RECORD_BYTES
    data = eeprom_read(addr, RECORD_BYTES)
    return struct.unpack('>ff', data)

# Initialise count if EEPROM is blank (count > MAX_RECORDS)
count = get_count()
if count > MAX_RECORDS:
    set_count(0); count = 0

REFRESH_MS = 10000
last_write = utime.ticks_ms() - REFRESH_MS
review_idx = -1
last_btn   = 0

lcd.clear(); lcd.putstr("EEPROM Logger"); utime.sleep(1)
lcd.clear(); lcd.putstr("Records: {:3d}".format(count))
utime.sleep(1)
print("Data logger active. {} records stored.".format(count))

while True:
    now = utime.ticks_ms()

    # Auto-log every 10 seconds
    if utime.ticks_diff(now, last_write) >= REFRESH_MS:
        try:
            dht_sensor.measure(); utime.sleep_ms(100)
            temp_c = dht_sensor.temperature()
            humid  = dht_sensor.humidity()
            slot   = count % MAX_RECORDS
            write_record(slot, temp_c, humid)
            if count < MAX_RECORDS:
                count += 1
                set_count(count)
            last_write = now
            print("Logged record {}: {:.1f}C {:.1f}%".format(slot, temp_c, humid))
            lcd.clear()
            lcd.putstr("Logged #{:2d}".format(slot))
            lcd.move_to(0, 1); lcd.putstr("T:{:.1f}C H:{:.0f}%".format(temp_c, humid))
        except OSError as e:
            print("DHT error:", e)

    # Button A: step through records
    if btn_review.value() == 0 and utime.ticks_diff(now, last_btn) > 400:
        last_btn   = now
        review_idx = (review_idx + 1) % max(1, count)
        if count > 0:
            t, h = read_record(review_idx)
            lcd.clear()
            lcd.putstr("Rec #{:2d}/{:2d}".format(review_idx, count - 1))
            lcd.move_to(0, 1); lcd.putstr("T:{:.1f}C H:{:.0f}%".format(t, h))
            print("Review #{}: {:.1f}C {:.1f}%".format(review_idx, t, h))

    # Button B: clear log
    if btn_clear.value() == 0 and utime.ticks_diff(now, last_btn) > 400:
        last_btn = now
        set_count(0); count = 0; review_idx = -1
        lcd.clear(); lcd.putstr("Log Cleared!")
        print("EEPROM log cleared.")
        utime.sleep(1)

    utime.sleep_ms(100)
```

## What to Click in MbedO
1. Drag **Pico**, **DHT22**, **AT24C32 EEPROM**, **two Buttons**, and **I2C LCD** onto the canvas.
2. Connect DHT22 to **GP16**, EEPROM and LCD to **GP4/GP5** (shared I2C). Connect Buttons to **GP13** and **GP14**. Connect power and GND.
3. Paste code, click **Run**. The first log entry appears after 10 seconds.
4. Press Button A to step through stored records. Press Button B to clear.

## Expected Output
```
Data logger active. 0 records stored.
Logged record 0: 24.5C 58.0%
Logged record 1: 24.6C 57.8%
Review #0: 24.5C 58.0%
EEPROM log cleared.
```

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `slot = count % MAX_RECORDS` | Implements a circular buffer: when 40 records are full, new records overwrite the oldest slot. |
| `eeprom_write(addr, struct.pack('>ff', temp, humid))` | Encodes two 32-bit IEEE 754 floats into 8 bytes for EEPROM storage. |

## Hardware & Safety Concept: EEPROM Write Endurance
AT24C32 EEPROMs are rated for ~1,000,000 write cycles per address. Logging every 10 seconds writes 8,640 records/day. To avoid wearing out EEPROM cells, always rotate writes across addresses using a circular buffer — never write to the same address repeatedly.

## Try This! (Challenges)
1. **RTC Timestamps**: Add a DS3231 RTC module on the same I2C bus and store the Unix timestamp alongside each record.
2. **OLED Graph**: After filling 16 records, plot temperature as a scrolling bar chart on an SSD1306 OLED.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| EEPROM read returns 0xFF | EEPROM never written | Verify address pins A0/A1/A2 are all tied to GND for address 0x50. |
| Records appear corrupted | Write cycle too fast | Ensure 10 ms delay after every `eeprom_write()` call. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [106 - Pico DHT22 Temperature Humidity LCD](../intermediate/106-pico-dht22-temperature-humidity-lcd.md)
- [140 - Pico Full Weather Console DHT22 BMP180 LDR OLED](140-pico-full-weather-console-dht22-bmp180-ldr-oled.md)
