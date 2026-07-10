# 135 - Pico RFID Locker System Servo LCD Buzzer

Build a multi-slot RFID locker system that authenticates user cards, assigns locker slots by UID, controls a servo latch, provides audible confirmation, and displays slot assignment and access status on an I2C LCD.

## Goal
Learn how to maintain a UID-to-locker mapping dictionary, authenticate RFID cards, toggle a servo latch on authorized access, provide audio confirmation tones, and display assignment status on an I2C character LCD in MicroPython.

## What You Will Build
A multi-slot RFID locker:
- **MFRC522 RFID (SPI: GP5-GP9)**: Reads user cards.
- **Servo Motor (GP10)**: Opens/closes the locker latch.
- **Green LED (GP13)**: Access granted indicator.
- **Red LED (GP14)**: Access denied indicator.
- **Passive Buzzer (GP15)**: Confirmation chime (access granted) or denial beep.
- **I2C 16x2 LCD (GP4, GP5)**: Displays user name, assigned locker, and lock state.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |
| Green LED | `led` | Yes | Yes |
| Red LED | `led` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 330 Ω Resistors × 2 | `resistor` | Optional in MbedO | Yes (one per LED) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 | SDA (SS) | GP5 | White | SPI Chip Select |
| MFRC522 | SCK | GP6 | Blue | SPI Clock |
| MFRC522 | MOSI | GP7 | Orange | SPI Master Out |
| MFRC522 | MISO | GP8 | Yellow | SPI Master In |
| MFRC522 | RST | GP9 | Purple | Hardware Reset |
| MFRC522 | VCC / GND | 3.3V / GND | Red / Black | Module power |
| Servo Motor | Control (PWM) | GP10 | Orange | Servo PWM signal |
| Servo Motor | Power (+) | 5V (VBUS) | Red | Servo power supply |
| Green LED | Anode (+, via 330 Ω) | GP13 | Green | Access granted indicator |
| Red LED | Anode (+, via 330 Ω) | GP14 | Red | Access denied indicator |
| Both LEDs | Cathode (−) | GND | Black | Shared ground return |
| Passive Buzzer | Signal (+) | GP15 | Grey | Tone output (PWM) |
| Passive Buzzer | GND (−) | GND | Black | Ground return |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** MFRC522 uses SPI (GP5-GP9). The LCD uses I2C Bus 0 on GP4/GP5. These are different buses on different pin sets and do not conflict. All grounds are shared.

## Code
```python
from machine import Pin, PWM, SPI, I2C
import utime
from mfrc522 import MFRC522
from machine_lcd import I2cLcd

spi  = SPI(0, sck=Pin(6), mosi=Pin(7), miso=Pin(8))
rfid = MFRC522(spi=spi, gpioRst=9, gpioCs=5)

servo     = PWM(Pin(10)); servo.freq(50)
led_green = Pin(13, Pin.OUT)
led_red   = Pin(14, Pin.OUT)
buzzer    = PWM(Pin(15)); buzzer.duty_u16(0)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Locker user database: UID (tuple) → (name, locker_number)
LOCKERS = {
    (0xDE, 0xAD, 0xBE, 0xEF): ("Alice", "L1"),
    (0xCA, 0xFE, 0xBA, 0xBE): ("Bob",   "L2"),
    (0x01, 0x02, 0x03, 0x04): ("Carol", "L3"),
}

# Latch state per locker
latch_open = {}  # locker_number → bool

def set_servo(angle):
    servo.duty_u16(1638 + int(angle * (8192 - 1638) / 180))

def tone(freq, ms):
    buzzer.freq(freq); buzzer.duty_u16(32768)
    utime.sleep_ms(ms); buzzer.duty_u16(0)

def granted_chime():
    tone(880, 120); utime.sleep_ms(60); tone(1047, 200)

def denied_beep():
    tone(300, 400)

set_servo(0)
led_green.value(0); led_red.value(0)

lcd.clear(); lcd.putstr("RFID Locker Sys")
utime.sleep(1)
print("RFID Locker System active. Scan your card.")

while True:
    rfid.init()
    (stat, _) = rfid.request(rfid.REQIDL)
    if stat == rfid.OK:
        (stat, raw_uid) = rfid.anticoll()
        if stat == rfid.OK:
            uid_tuple = tuple(raw_uid[:4])
            print("Scanned UID:", [hex(b) for b in uid_tuple])

            if uid_tuple in LOCKERS:
                name, locker = LOCKERS[uid_tuple]
                # Toggle latch for this locker
                is_open = latch_open.get(locker, False)
                new_open = not is_open
                latch_open[locker] = new_open

                set_servo(90 if new_open else 0)
                led_green.value(1); led_red.value(0)
                granted_chime()
                utime.sleep_ms(300)
                led_green.value(0)

                print(">> {} → {} {}".format(name, locker, "OPEN" if new_open else "CLOSED"))

                lcd.clear()
                lcd.move_to(0, 0)
                lcd.putstr("User: " + name[:10])
                lcd.move_to(0, 1)
                lcd.putstr("{}: {}".format(locker, "OPEN  " if new_open else "CLOSED"))

            else:
                led_red.value(1); led_green.value(0)
                denied_beep()
                utime.sleep_ms(300)
                led_red.value(0)
                print(">> DENIED: UID not registered.")

                lcd.clear()
                lcd.move_to(0, 0)
                lcd.putstr("ACCESS DENIED")
                lcd.move_to(0, 1)
                lcd.putstr("Unknown Card")

            utime.sleep(1)  # Anti-bounce delay

    else:
        lcd.clear()
        lcd.putstr("RFID Locker Sys")
        lcd.move_to(0, 1)
        lcd.putstr("Scan card...    ")

    utime.sleep_ms(100)
```

## What to Click in MbedO
1. Drag **Pico**, **MFRC522**, **Servo**, **Green LED**, **Red LED**, **Passive Buzzer**, and **I2C LCD** onto the canvas.
2. Wire per the wiring table. All grounds shared.
3. Update `LOCKERS` with your card UIDs (scan first to reveal UID in serial output). Paste code, select **MicroPython**, click **Run**.
4. Scan an authorized card to open/close the latch. Scan an unknown card for a denial beep.

## Expected Output
```
RFID Locker System active. Scan your card.
Scanned UID: ['0xde', '0xad', '0xbe', '0xef']
>> Alice → L1 OPEN
Scanned UID: ['0xde', '0xad', '0xbe', '0xef']
>> Alice → L1 CLOSED
```

## Expected Canvas Behavior
- Authorized card scan: servo sweeps to 90° (open), green LED lights, and chime plays. LCD shows user name and locker state. Second scan toggles back to 0° (closed). Unknown card: red LED + denial beep.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `LOCKERS = {(0xDE,...): ("Alice", "L1")}` | Uses UID tuples as dictionary keys to look up user name and locker number. |
| `latch_open.get(locker, False)` | Retrieves the current latch state for this locker, defaulting to closed if not yet set. |

## Hardware & Safety Concept: UID-Based Access Control
Real access control systems store UID-to-user mappings in a database server, not on-device. The reader sends the UID to the server, which validates it against a live roster and logs the event. On-device databases (as used here) are simpler but cannot be remotely revoked or audited.

## Try This! (Challenges)
1. **Admin Override**: Add a special master card UID that unlocks all lockers simultaneously.
2. **Auto-Close Timer**: After opening, automatically close the servo after 10 seconds using `utime.ticks_ms()`.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer produces no tone | Active buzzer connected | Use a **passive** buzzer to produce variable PWM tones. |
| Card UID not found | UID format mismatch | Print `tuple(raw_uid[:4])` and copy the exact byte values into the LOCKERS dictionary. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [96 - Pico RFID Access Control LCD](../intermediate/96-pico-rfid-access-control-lcd.md)
- [121 - Pico RFID PIR Smart Lock LCD](121-pico-rfid-pir-smart-lock-lcd.md)
