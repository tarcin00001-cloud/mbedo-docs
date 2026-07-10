# 137 - Pico Keypad RFID Dual Auth System

Build a high-security dual-authentication access controller that requires BOTH a valid RFID card AND a correct PIN entered on a matrix keypad before unlocking a servo latch, providing two-factor authentication on embedded hardware.

## Goal
Learn how to implement two-factor authentication (2FA) on a microcontroller: RFID card verification as factor 1, followed by keypad PIN verification as factor 2, with a timeout that resets the authentication state if either factor is not completed quickly enough.

## What You Will Build
A dual-factor access controller:
- **MFRC522 RFID (SPI: GP5-GP9)**: Factor 1 — card must be authenticated first.
- **4×4 Matrix Keypad (GP0-GP3 rows, GP10-GP13 cols)**: Factor 2 — PIN must be entered within 10 seconds.
- **Servo Motor (GP11)**: Unlocks at 90° on dual-factor success, locks at 0° otherwise.
- **Green LED (GP14)**: Access granted indicator.
- **Red LED (GP15)**: Access denied / locked indicator.
- **I2C 16x2 LCD (GP4, GP5)**: Guides the user through each authentication stage.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| 4×4 Matrix Keypad | `keypad` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |
| Green LED | `led` | Yes | Yes |
| Red LED | `led` | Yes | Yes |
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
| Keypad Rows | R1–R4 | GP0, GP1, GP2, GP3 | Blue wires | Row scan lines |
| Keypad Cols | C1–C4 | GP10, GP11, GP12, GP13 | Yellow wires | Column scan lines |
| Servo Motor | Control (PWM) | GP11 | Orange | NOTE: use GP20 to avoid keypad col conflict — see note below |
| Servo Motor | Power (+) | 5V (VBUS) | Red | Servo power supply |
| Green LED | Anode (+, via 330 Ω) | GP14 | Green | Granted indicator |
| Red LED | Anode (+, via 330 Ω) | GP15 | Red | Denied/locked indicator |
| Both LEDs | Cathode (−) | GND | Black | Shared ground return |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Move the servo to **GP20** to avoid sharing GP11 with keypad column C2. The MFRC522 uses SPI (GP5-GP9). The LCD uses I2C Bus 0 (GP4/GP5). Keypad rows use GP0-GP3 and columns use GP10-GP13. All grounds are shared.

## Code
```python
from machine import Pin, PWM, SPI, I2C
import utime
from mfrc522 import MFRC522
from machine_lcd import I2cLcd

# MFRC522 RFID
spi  = SPI(0, sck=Pin(6), mosi=Pin(7), miso=Pin(8))
rfid = MFRC522(spi=spi, gpioRst=9, gpioCs=5)

# Servo on GP20
servo = PWM(Pin(20)); servo.freq(50)

led_green = Pin(14, Pin.OUT)
led_red   = Pin(15, Pin.OUT)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Keypad matrix
ROWS = [Pin(p, Pin.OUT) for p in (0, 1, 2, 3)]
COLS = [Pin(p, Pin.IN, Pin.PULL_DOWN) for p in (10, 11, 12, 13)]
KEYS = [
    ['1','2','3','A'],
    ['4','5','6','B'],
    ['7','8','9','C'],
    ['*','0','#','D'],
]

# Auth configuration
AUTHORIZED_UID = (0xDE, 0xAD, 0xBE, 0xEF)
SECRET_PIN     = "1234"
PIN_TIMEOUT_S  = 10

def set_servo(angle):
    servo.duty_u16(1638 + int(angle * (8192 - 1638) / 180))

def scan_keypad():
    for r, row_pin in enumerate(ROWS):
        row_pin.value(1)
        for c, col_pin in enumerate(COLS):
            if col_pin.value() == 1:
                row_pin.value(0)
                utime.sleep_ms(50)  # Debounce
                return KEYS[r][c]
        row_pin.value(0)
    return None

# States
WAITING_CARD = 0
WAITING_PIN  = 1
UNLOCKED     = 2

state = WAITING_CARD
set_servo(0)
led_red.value(1); led_green.value(0)

lcd.clear(); lcd.putstr("2FA Access Ctrl")
utime.sleep(1)

card_verified = False
pin_entry     = ""
pin_timer     = 0

print("Dual-factor auth system active. Scan card first.")

while True:
    if state == WAITING_CARD:
        led_red.value(1); led_green.value(0)
        lcd.clear(); lcd.putstr("Step 1: Scan"); lcd.move_to(0, 1); lcd.putstr("your card...    ")

        rfid.init()
        (s, _) = rfid.request(rfid.REQIDL)
        if s == rfid.OK:
            (s, uid) = rfid.anticoll()
            if s == rfid.OK and tuple(uid[:4]) == AUTHORIZED_UID:
                state     = WAITING_PIN
                pin_entry = ""
                pin_timer = utime.ticks_ms()
                print(">> Card OK. Enter PIN.")
            else:
                lcd.clear(); lcd.putstr("CARD DENIED")
                led_red.value(1)
                utime.sleep(1)

    elif state == WAITING_PIN:
        elapsed = utime.ticks_diff(utime.ticks_ms(), pin_timer) // 1000
        remain  = max(0, PIN_TIMEOUT_S - elapsed)

        if remain == 0:
            state = WAITING_CARD
            print(">> PIN timeout. Restart.")
            continue

        lcd.clear(); lcd.putstr("Step 2: PIN ({:2d}s)".format(remain))
        lcd.move_to(0, 1); lcd.putstr(">" + "*" * len(pin_entry) + "          ")

        key = scan_keypad()
        if key:
            if key == '#':
                # Confirm PIN
                if pin_entry == SECRET_PIN:
                    state = UNLOCKED
                    set_servo(90)
                    led_green.value(1); led_red.value(0)
                    print(">> ACCESS GRANTED. Unlocked.")
                else:
                    state = WAITING_CARD
                    pin_entry = ""
                    lcd.clear(); lcd.putstr("WRONG PIN"); led_red.value(1)
                    utime.sleep(1)
                    print(">> Wrong PIN. Restart.")
            elif key == '*':
                pin_entry = ""  # Clear
            else:
                pin_entry += key
                if len(pin_entry) > 8:
                    pin_entry = pin_entry[-8:]

        utime.sleep_ms(150)

    elif state == UNLOCKED:
        lcd.clear(); lcd.putstr("ACCESS GRANTED"); lcd.move_to(0, 1); lcd.putstr("Scan=Lock       ")
        # Wait for card scan to re-lock
        rfid.init()
        (s, _) = rfid.request(rfid.REQIDL)
        if s == rfid.OK:
            state = WAITING_CARD
            set_servo(0)
            led_green.value(0); led_red.value(1)
            print(">> Locked.")
        utime.sleep_ms(200)
```

## What to Click in MbedO
1. Drag **Pico**, **MFRC522**, **Keypad**, **Servo**, **Green LED**, **Red LED**, and **I2C LCD** onto the canvas.
2. Wire per the wiring table. Move servo to GP20.
3. Paste code, update `AUTHORIZED_UID` and `SECRET_PIN`. Click **Run**.
4. Scan the authorized card, then type `1234#` on the keypad to unlock. Scan card again to re-lock.

## Expected Output
```
Dual-factor auth system active. Scan card first.
>> Card OK. Enter PIN.
>> ACCESS GRANTED. Unlocked.
>> Locked.
```

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| State machine `WAITING_CARD → WAITING_PIN → UNLOCKED` | Each stage gates the next — the PIN entry stage only becomes reachable after a valid card scan. |
| `remain == 0 → state = WAITING_CARD` | PIN entry timeout forces a restart from card scan, preventing brute-force PIN guessing. |

## Hardware & Safety Concept: Two-Factor Authentication (2FA)
2FA requires two independent proofs of identity from different categories: **something you have** (RFID card) and **something you know** (PIN). Compromising one factor alone is insufficient for access. A stolen card without the PIN, or a known PIN without the card, both fail authentication — the same principle used in bank EMV chip-and-PIN cards.

## Try This! (Challenges)
1. **Lockout Counter**: After 3 wrong PIN attempts, lock the system for 60 seconds and display a "LOCKED OUT" message.
2. **Admin Master**: Add a second authorized card UID that bypasses the PIN requirement (admin override).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Keypad does not respond | Row/column pin swap | Swap ROW and COL assignments — some keypads have inverted row/column ordering. |
| Servo twitches during keypad scan | GP11 shared with keypad col | Move servo PWM to GP20 as noted in the wiring table. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [97 - Pico Keypad Lock LCD](../intermediate/97-pico-keypad-lock-lcd.md)
- [135 - Pico RFID Locker System Servo LCD Buzzer](135-pico-rfid-locker-system-servo-lcd-buzzer.md)
