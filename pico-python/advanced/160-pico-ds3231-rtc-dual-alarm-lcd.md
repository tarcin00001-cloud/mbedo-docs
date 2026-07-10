# 160 - Pico Real-Time Clock DS3231 Alarm LCD

Build a precision real-time clock display and dual-alarm system using a DS3231 RTC module, display the time and date on an I2C LCD, allow setting two daily alarms via keypad, and trigger a buzzer and relay when an alarm fires.

## Goal
Learn how to communicate with a DS3231 RTC over I2C, read and write time/date registers in BCD format, configure alarm registers, and detect alarm flags to trigger outputs in MicroPython.

## What You Will Build
A dual-alarm real-time clock:
- **DS3231 RTC (GP4, GP5 I2C)**: Maintains accurate time and triggers alarm flags.
- **I2C 16x2 LCD (GP4, GP5 shared bus)**: Displays current time, date, and alarm state.
- **4-button Keypad (GP13-GP16)**: Set hours/minutes for the active alarm.
- **Buzzer (GP10)**: Sounds when an alarm fires.
- **Relay (GP11)**: Could drive a lamp or coffee machine on alarm.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DS3231 RTC Module | `ds3231` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| Tactile Push Buttons × 4 | `button` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS3231 | VCC | 3.3V (3V3) | Red | Module power |
| DS3231 | GND | GND | Black | Ground reference |
| DS3231 | SDA | GP4 | Orange | Shared I2C Bus 0 |
| DS3231 | SCL | GP5 | Blue | Shared I2C Bus 0 |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| Button SELECT | Terminal 1 / 2 | GP13 / GND | White / Black | Select alarm A/B |
| Button HOUR+ | Terminal 1 / 2 | GP14 / GND | White / Black | Increase alarm hour |
| Button MIN+ | Terminal 1 / 2 | GP15 / GND | White / Black | Increase alarm minute |
| Button SNOOZE | Terminal 1 / 2 | GP16 / GND | White / Black | Snooze / dismiss alarm |
| Buzzer | + | GP10 | Orange | Alarm sound output |
| Relay | IN | GP11 | Purple | Appliance relay control |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** DS3231 and LCD share I2C Bus 0 (GP4/GP5). DS3231 default address is 0x68. Buttons on GP13-GP16 with PULL_UP. Buzzer on GP10, relay on GP11. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime
from machine_lcd import I2cLcd

i2c    = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd    = I2cLcd(i2c, 0x27, 2, 16)
buzzer = Pin(10, Pin.OUT); buzzer.value(0)
relay  = Pin(11, Pin.OUT); relay.value(0)

btn_sel   = Pin(13, Pin.IN, Pin.PULL_UP)
btn_hr    = Pin(14, Pin.IN, Pin.PULL_UP)
btn_min   = Pin(15, Pin.IN, Pin.PULL_UP)
btn_snz   = Pin(16, Pin.IN, Pin.PULL_UP)

DS3231_ADDR = 0x68

def bcd_to_dec(bcd): return (bcd >> 4) * 10 + (bcd & 0x0F)
def dec_to_bcd(dec): return ((dec // 10) << 4) | (dec % 10)

def rtc_read_time():
    data = i2c.readfrom_mem(DS3231_ADDR, 0x00, 7)
    sec  = bcd_to_dec(data[0] & 0x7F)
    mn   = bcd_to_dec(data[1])
    hr   = bcd_to_dec(data[2] & 0x3F)
    day  = bcd_to_dec(data[4])
    mon  = bcd_to_dec(data[5] & 0x1F)
    yr   = 2000 + bcd_to_dec(data[6])
    return hr, mn, sec, day, mon, yr

def rtc_set_time(hr, mn, sec, day, mon, yr):
    i2c.writeto_mem(DS3231_ADDR, 0x00, bytes([
        dec_to_bcd(sec), dec_to_bcd(mn), dec_to_bcd(hr),
        0x01,            dec_to_bcd(day), dec_to_bcd(mon),
        dec_to_bcd(yr - 2000)
    ]))

# Alarm settings (software-based for simplicity)
alarms = [
    {'hr': 7, 'mn': 30, 'enabled': True,  'label': "A1"},
    {'hr': 12, 'mn': 0, 'enabled': False, 'label': "A2"},
]
active_alarm = 0
snooze_until = None
alarm_firing = False
last_btn     = 0

# Boot: set time if RTC is in year 2000 (uninitialised)
h, m, s, d, mo, y = rtc_read_time()
if y == 2000:
    rtc_set_time(8, 0, 0, 10, 7, 26)
    print("RTC initialised to 08:00:00 2026-07-10")

print("RTC alarm clock active.")
lcd.clear(); lcd.putstr("RTC Alarm Clock"); utime.sleep(1)

last_sec = -1

while True:
    now = utime.ticks_ms()
    hr, mn, sec, day, mon, yr = rtc_read_time()

    # Check alarms (only once per minute)
    if sec == 0 and sec != last_sec and not alarm_firing:
        for al in alarms:
            if al['enabled'] and al['hr'] == hr and al['mn'] == mn:
                if snooze_until is None or utime.ticks_diff(now, snooze_until) >= 0:
                    alarm_firing = True
                    print("ALARM:", al['label'])
    last_sec = sec

    # Alarm outputs
    if alarm_firing:
        buzzer.value(now // 500 % 2)
        relay.value(1)
    else:
        buzzer.value(0)
        relay.value(0)

    # --- Buttons ---
    if btn_sel.value() == 0 and utime.ticks_diff(now, last_btn) > 400:
        active_alarm = (active_alarm + 1) % 2
        last_btn = now
        print("Active alarm:", alarms[active_alarm]['label'])

    if btn_hr.value() == 0 and utime.ticks_diff(now, last_btn) > 300:
        alarms[active_alarm]['hr'] = (alarms[active_alarm]['hr'] + 1) % 24
        last_btn = now

    if btn_min.value() == 0 and utime.ticks_diff(now, last_btn) > 300:
        alarms[active_alarm]['mn'] = (alarms[active_alarm]['mn'] + 1) % 60
        last_btn = now

    if btn_snz.value() == 0 and utime.ticks_diff(now, last_btn) > 400:
        last_btn = now
        if alarm_firing:
            alarm_firing  = False
            snooze_until  = now + 5 * 60 * 1000  # 5 min snooze
            print("Snooze 5 min.")
        else:
            alarms[active_alarm]['enabled'] = not alarms[active_alarm]['enabled']
            print("Alarm {} enabled:{}".format(
                alarms[active_alarm]['label'], alarms[active_alarm]['enabled']))

    # --- LCD ---
    al = alarms[active_alarm]
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("{:02d}:{:02d}:{:02d} {:02d}/{:02d}".format(hr, mn, sec, day, mon))
    lcd.move_to(0, 1)
    en = "ON " if al['enabled'] else "OFF"
    fire = "!!" if alarm_firing else "  "
    lcd.putstr("{}{} {:02d}:{:02d} {}".format(al['label'], fire, al['hr'], al['mn'], en))

    utime.sleep_ms(200)
```

## What to Click in MbedO
1. Drag **Pico**, **DS3231 RTC**, **I2C LCD**, **four Buttons**, **Buzzer**, and **Relay** onto the canvas.
2. Connect DS3231 and LCD to **GP4/GP5**. Connect Buttons to **GP13-GP16**. Buzzer to **GP10**, Relay to **GP11**. Connect power and GND.
3. Paste code, click **Run**. The LCD shows current time. Press SELECT to switch between A1/A2. Press HOUR+/MIN+ to set alarm time. Press SNOOZE to enable/disable.

## Expected Output
```
RTC alarm clock active.
Active alarm: A2
ALARM: A1
Snooze 5 min.
```

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `bcd_to_dec(bcd)` | DS3231 stores time as Binary Coded Decimal — each nibble holds one decimal digit. This converts it to plain integers. |
| `buzzer.value(now // 500 % 2)` | Non-blocking 1 Hz buzzer tone: toggles every 500 ms using the millisecond tick. |

## Hardware & Safety Concept: DS3231 Temperature-Compensated Crystal
The DS3231 uses an internal MEMS temperature-compensated crystal oscillator (TCXO) that actively corrects the oscillator frequency for temperature drift. This gives ±2 ppm accuracy (±1 minute/year) across -40°C to +85°C — far more precise than the DS1307 which uses an external 32 kHz crystal (±20 ppm).

## Try This! (Challenges)
1. **OLED Clock Face**: Render an analogue clock face with hour, minute, and second hands on an SSD1306 OLED using trigonometry.
2. **Weekly Alarm**: Extend the alarm structure to include a day-of-week mask, allowing alarms only on weekdays or weekends.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Time resets on every boot | RTC backup battery dead | Replace the DS3231 module's CR2032 coin cell battery. |
| Clock reads 00:00:00 year 2000 | RTC never initialised | The code auto-sets time at boot if year is 2000; adjust the initial time in `rtc_set_time()`. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [141 - Pico I2C EEPROM Data Logger DHT22 LCD](141-pico-i2c-eeprom-data-logger-dht22-lcd.md)
- [155 - Pico Environmental Data Station SD Card Logger](155-pico-environmental-data-station-sd-card-logger.md)
