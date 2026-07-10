# 150 - Pico Full Alarm Panel Keypad RFID LCD

Build a comprehensive four-zone alarm panel that combines RFID arm/disarm authentication, a keypad PIN override, independent zone monitoring (PIR, door, vibration, glass-break simulation), and a multi-output alert system with full LCD status display.

## Goal
Learn how to build a state-machine alarm panel with multiple authentication methods (RFID or PIN), four independent sensor zones, zone bypass capability, and a graduated alarm output system on an I2C LCD in MicroPython.

## What You Will Build
A four-zone alarm panel:
- **MFRC522 RFID (SPI GP5-GP9)**: Primary arm/disarm authentication.
- **4×3 Keypad (GP0-GP2 rows, GP10-GP12 cols)**: PIN override for arm/disarm.
- **Zone 1 PIR (GP13)**: Perimeter motion sensor.
- **Zone 2 Door Sensor (GP14)**: Magnetic door contact.
- **Zone 3 Vibration (GP15)**: Vibration sensor.
- **Zone 4 Button (GP16)**: Simulates glass-break sensor.
- **Siren Relay (GP17)**: External alarm siren.
- **LED (GP18)**: Armed status indicator.
- **I2C 16x2 LCD (GP4, GP5)**: Zone status, armed state, and alert display.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| 4×3 Matrix Keypad | `keypad` | Yes (or 4×4) | Yes |
| PIR / Buttons × 4 | `button` | Yes | Yes |
| Relay Module (Siren) | `relay` | Yes | Yes |
| LED | `led` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 | SDA/SCK/MOSI/MISO/RST | GP5/GP6/GP7/GP8/GP9 | Mixed | SPI bus for RFID |
| MFRC522 | VCC / GND | 3.3V / GND | Red / Black | RFID module power |
| Keypad Rows | R1-R3 | GP0, GP1, GP2 | Blue wires | Row scan output pins |
| Keypad Cols | C1-C3 | GP10, GP11, GP12 | Yellow wires | Column read inputs |
| Zone 1 PIR | OUT | GP13 | Blue | Motion detection input |
| Zone 2 Door | OUT | GP14 | Green | Door contact input |
| Zone 3 Vibration | OUT | GP15 | Orange | Vibration sensor input |
| Zone 4 Glass-Break | OUT | GP16 | Red | Glass sensor simulation |
| Siren Relay | IN | GP17 | Purple | External siren control |
| Armed LED | Anode (via 330 Ω) | GP18 | Red | Armed state indicator |
| Armed LED | Cathode | GND | Black | Ground return |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** RFID uses SPI on GP5-GP9. Zone sensors connect to GP13-GP16 with PULL_DOWN. Keypad rows GP0-GP2 and cols GP10-GP12. Siren relay GP17. Armed LED GP18. LCD on GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, SPI, I2C
import utime
from mfrc522 import MFRC522
from machine_lcd import I2cLcd

# RFID
spi  = SPI(0, sck=Pin(6), mosi=Pin(7), miso=Pin(8))
rfid = MFRC522(spi=spi, gpioRst=9, gpioCs=5)

# Keypad (3×3 subset of 4×3)
ROWS = [Pin(p, Pin.OUT) for p in (0, 1, 2)]
COLS = [Pin(p, Pin.IN, Pin.PULL_DOWN) for p in (10, 11, 12)]
KEYS = [['1','2','3'],['4','5','6'],['7','8','#']]

# Zone sensors (active HIGH)
ZONES = [
    Pin(13, Pin.IN, Pin.PULL_DOWN),  # Z1 PIR
    Pin(14, Pin.IN, Pin.PULL_DOWN),  # Z2 Door
    Pin(15, Pin.IN, Pin.PULL_DOWN),  # Z3 Vibration
    Pin(16, Pin.IN, Pin.PULL_DOWN),  # Z4 Glass-Break
]
ZONE_NAMES = ["PIR ", "DOOR", "VIB ", "GLAS"]

siren     = Pin(17, Pin.OUT)
armed_led = Pin(18, Pin.OUT)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

siren.value(0); armed_led.value(0)

AUTH_UID = (0xDE, 0xAD, 0xBE, 0xEF)
PIN_CODE = "1234"

# States
DISARMED = 0; ARMED = 1; ALARM = 2
state      = DISARMED
pin_buf    = ""
alarm_zone = None

def scan_key():
    for r, rp in enumerate(ROWS):
        rp.value(1)
        for c, cp in enumerate(COLS):
            if cp.value():
                rp.value(0)
                utime.sleep_ms(40)
                return KEYS[r][c]
        rp.value(0)
    return None

def scan_rfid():
    rfid.init()
    s, _ = rfid.request(rfid.REQIDL)
    if s == rfid.OK:
        s, uid = rfid.anticoll()
        if s == rfid.OK and tuple(uid[:4]) == AUTH_UID:
            return True
    return False

def check_zones():
    for i, z in enumerate(ZONES):
        if z.value():
            return i
    return None

def update_lcd():
    lcd.clear()
    lcd.move_to(0, 0)
    if state == DISARMED:
        lcd.putstr("DISARMED        ")
    elif state == ARMED:
        lcd.putstr("ARMED  Ready    ")
    else:
        lcd.putstr("!! ALARM Z{} !!  ".format(alarm_zone + 1 if alarm_zone is not None else "?"))
    lcd.move_to(0, 1)
    states = " ".join(ZONE_NAMES[i] + ("!" if ZONES[i].value() else ".") for i in range(4))
    lcd.putstr(states[:16])

lcd.clear(); lcd.putstr("Alarm Panel"); utime.sleep(1)
print("Alarm panel active. RFID or PIN to arm/disarm.")

last_btn = 0

while True:
    now = utime.ticks_ms()

    # --- RFID authentication ---
    if scan_rfid():
        if state == DISARMED:
            state = ARMED; armed_led.value(1)
            print("ARMED via RFID.")
        elif state in (ARMED, ALARM):
            state = DISARMED; armed_led.value(0)
            siren.value(0); alarm_zone = None
            print("DISARMED via RFID.")
        utime.sleep(1)

    # --- Keypad PIN ---
    key = scan_key()
    if key and utime.ticks_diff(now, last_btn) > 250:
        last_btn = now
        if key == '#':
            if pin_buf == PIN_CODE:
                if state == DISARMED:
                    state = ARMED; armed_led.value(1); print("ARMED via PIN.")
                else:
                    state = DISARMED; armed_led.value(0)
                    siren.value(0); alarm_zone = None; print("DISARMED via PIN.")
            else:
                print("Wrong PIN:", pin_buf)
            pin_buf = ""
        else:
            pin_buf += key
            if len(pin_buf) > 6: pin_buf = pin_buf[-6:]

    # --- Zone monitoring ---
    if state == ARMED:
        triggered = check_zones()
        if triggered is not None:
            state = ALARM; alarm_zone = triggered
            siren.value(1)
            print("ALARM! Zone {} ({}) triggered.".format(
                triggered + 1, ZONE_NAMES[triggered]))

    update_lcd()

    # Siren pulse in alarm state
    if state == ALARM:
        siren.value(1); utime.sleep_ms(300)
        siren.value(0); utime.sleep_ms(300)
    else:
        utime.sleep_ms(100)
```

## What to Click in MbedO
1. Drag **Pico**, **MFRC522**, **Keypad**, **four Zone Buttons**, **Relay**, **LED**, and **I2C LCD** onto the canvas.
2. Wire per the wiring table. All grounds shared.
3. Paste code, update `AUTH_UID`. Click **Run**.
4. Scan authorized card or type `1234#` to arm. Click a Zone button while armed to trigger alarm. Scan card or type PIN to disarm.

## Expected Output
```
Alarm panel active. RFID or PIN to arm/disarm.
ARMED via PIN.
ALARM! Zone 1 (PIR ) triggered.
DISARMED via RFID.
```

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| State machine `DISARMED → ARMED → ALARM` | Zone checking only occurs in the ARMED state; transition to ALARM is one-way until authenticated disarm. |
| `pin_buf += key` | Accumulates keypad digits into a string buffer, compared against `PIN_CODE` on `#` press. |

## Hardware & Safety Concept: Dual-Path Authentication
Professional alarm panels support multiple authentication paths (card, PIN, remote) so that a failure of one method does not lock out the system. The RFID and PIN paths in this project are independent: either path can arm or disarm, providing redundancy.

## Try This! (Challenges)
1. **Entry Delay**: Add a 10-second countdown after arming before zone monitoring begins, allowing the user to exit through the monitored door.
2. **Zone Bypass**: Add key `*` + zone number to bypass a specific zone (mark it in a bypass list and skip it in `check_zones()`).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers immediately on arm | Zone sensor active at rest | Check zone sensor wiring — ensure pull-down is connected and sensor is normally-open. |
| PIN never accepted | Key bounce counting extra digits | Increase debounce delay to 300 ms. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [137 - Pico Keypad RFID Dual Auth System](137-pico-keypad-rfid-dual-auth-system.md)
- [133 - Pico Autonomous Guard HC-SR04 PIR Buzzer LCD](133-pico-autonomous-guard-hcsr04-pir-buzzer-lcd.md)
