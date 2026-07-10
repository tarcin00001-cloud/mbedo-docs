# 121 - Pico RFID PIR Smart Lock LCD

Build a smart lock that requires RFID card authentication to arm/disarm a PIR motion alarm. After authentication, a servo unlocks the door, PIR monitors for intruders, and a multi-state LCD shows the full security status.

## Goal
Learn how to chain RFID authentication as a gate before a PIR alarm zone becomes active, control a servo lock mechanism, manage multiple output states (LED, buzzer, relay), and display full system status on an I2C LCD in MicroPython.

## What You Will Build
A multi-stage smart lock:
- **MFRC522 RFID (SPI: GP5-GP9)**: Reads cards. Authorized card arms/disarms the system.
- **PIR Sensor (GP10)**: Monitors for motion ONLY when the system is armed.
- **Servo Motor (GP11)**: Unlocks (90°) on authorized card; locks (0°) when armed again.
- **Green LED (GP13)**: ON = unlocked/disarmed.
- **Red LED (GP14)**: ON = armed/locked.
- **Buzzer (GP15)**: Short chime on authorized; alarm siren on PIR trigger.
- **I2C 16x2 LCD (GP4, GP5)**: Displays full system state.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| PIR Motion Sensor | `button` | Yes (digital output) | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |
| Green LED | `led` | Yes | Yes |
| Red LED | `led` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
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
| MFRC522 | VCC | 3.3V (3V3) | Red | Module power |
| MFRC522 | GND | GND | Black | Ground reference |
| PIR Sensor | OUT | GP10 | Blue | Motion detector output (HIGH on motion) |
| PIR Sensor | VCC | 5V (VBUS) | Red | PIR power |
| PIR Sensor | GND | GND | Black | Ground reference |
| Servo Motor | Control (PWM) | GP11 | Orange | Servo PWM signal |
| Servo Motor | VCC (+) | 5V (VBUS) | Red | Servo power supply |
| Green LED | Anode (+, via 330 Ω) | GP13 | Green | Unlocked/disarmed indicator |
| Red LED | Anode (+, via 330 Ω) | GP14 | Red | Armed/locked indicator |
| Active Buzzer | VCC (+) | GP15 | Red | Alarm/chime output |
| All Modules | GND | GND | Black | Shared return path to ground |
| I2C LCD | SDA / SCL | GP4 / GP5 | Orange/Blue | Shared I2C screen bus |

> **Wiring tip:** Note that the MFRC522 SDA (chip select) uses GP5 while the I2C LCD SDA also uses GP4 (I2C bus 0). These are different bus types (SPI vs I2C) so they do not conflict. All grounds are shared.

## Code
```python
from machine import Pin, PWM, SPI, I2C
import utime
from mfrc522 import MFRC522
from machine_lcd import I2cLcd

# Initialize SPI and RFID reader
spi  = SPI(0, sck=Pin(6), mosi=Pin(7), miso=Pin(8))
rfid = MFRC522(spi=spi, gpioRst=9, gpioCs=5)

pir       = Pin(10, Pin.IN, Pin.PULL_DOWN)
servo_pwm = PWM(Pin(11))
servo_pwm.freq(50)

led_green = Pin(13, Pin.OUT)
led_red   = Pin(14, Pin.OUT)
buzzer    = Pin(15, Pin.OUT)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(4), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Authorized card UID (update this to your card's UID)
AUTHORIZED_UID = [0xDE, 0xAD, 0xBE, 0xEF]

def set_servo(angle):
    """Set servo angle: 0° = locked, 90° = unlocked."""
    duty = 1638 + int(angle * (8192 - 1638) / 180)
    servo_pwm.duty_u16(duty)

def chime(freq, ms):
    from machine import PWM as _PWM
    # Simple tone using the buzzer pin as digital toggle (active buzzer)
    buzzer.value(1)
    utime.sleep_ms(ms)
    buzzer.value(0)

# System states
LOCKED   = "LOCKED"
UNLOCKED = "UNLOCKED"
ALARM    = "ALARM"

state = LOCKED
set_servo(0)    # Servo → locked position
led_red.value(1)
led_green.value(0)
buzzer.value(0)

lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("Smart Lock Ready")
lcd.move_to(0, 1)
lcd.putstr("Scan Card...")
utime.sleep(2)

print("Smart lock system active. State:", state)

while True:
    # --- RFID Scan ---
    rfid.init()
    (stat, tag_type) = rfid.request(rfid.REQIDL)
    if stat == rfid.OK:
        (stat, raw_uid) = rfid.anticoll()
        if stat == rfid.OK:
            uid_list = list(raw_uid[:4])
            
            if uid_list == AUTHORIZED_UID:
                # Toggle between LOCKED and UNLOCKED
                if state == LOCKED:
                    state = UNLOCKED
                    set_servo(90)  # Open lock
                    led_green.value(1)
                    led_red.value(0)
                    chime(880, 100)
                    print(">> ACCESS GRANTED. State: UNLOCKED")
                elif state in (UNLOCKED, ALARM):
                    state = LOCKED
                    set_servo(0)   # Lock
                    led_green.value(0)
                    led_red.value(1)
                    buzzer.value(0) # Silence alarm
                    chime(660, 100)
                    print(">> SYSTEM ARMED. State: LOCKED")
            else:
                buzzer.value(1)
                utime.sleep_ms(500)
                buzzer.value(0)
                print(">> ACCESS DENIED. UID:", [hex(b) for b in uid_list])
                
    # --- PIR Monitoring (only when LOCKED / armed) ---
    if state == LOCKED and pir.value() == 1:
        state = ALARM
        print(">> INTRUDER DETECTED. State: ALARM")
        
    # --- Pulsing alarm when triggered ---
    if state == ALARM:
        led_red.value(1)
        buzzer.value(1)
        utime.sleep_ms(300)
        buzzer.value(0)
        utime.sleep_ms(300)
    else:
        utime.sleep_ms(100)
        
    # --- LCD Update ---
    lcd.clear()
    lcd.move_to(0, 0)
    if state == LOCKED:
        lcd.putstr("Status: LOCKED")
    elif state == UNLOCKED:
        lcd.putstr("Status: OPEN")
    else:
        lcd.putstr("!! ALARM !!")
    lcd.move_to(0, 1)
    if state == ALARM:
        lcd.putstr("Scan card=Reset")
    elif state == UNLOCKED:
        lcd.putstr("Scan=Relock")
    else:
        lcd.putstr("Scan card=Unlock")
```

## What to Click in MbedO
1. Drag all components onto the canvas: **Pico**, **MFRC522**, **PIR Button**, **Servo**, **Green LED**, **Red LED**, **Active Buzzer**, **I2C LCD**.
2. Wire them according to the wiring table. Connect all grounds.
3. Update `AUTHORIZED_UID` with your card's UID (printed in the serial output when you scan it first).
4. Paste the code, select **MicroPython** mode, and click **Run**.
5. Scan the authorized card to unlock. Scan again to lock. Click the PIR button while locked to trigger the intruder alarm.

## Expected Output
```
Smart lock system active. State: LOCKED
>> ACCESS GRANTED. State: UNLOCKED
>> SYSTEM ARMED. State: LOCKED
>> INTRUDER DETECTED. State: ALARM
>> ACCESS GRANTED. State: UNLOCKED
```

## Expected Canvas Behavior
- The servo sweeps to 90° when unlocked. The green/red LEDs switch roles. The buzzer and red LED pulse continuously in the ALARM state. Scanning the authorized card in ALARM state resets to UNLOCKED.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `state == LOCKED and pir.value() == 1` | PIR is only evaluated when the system is armed (LOCKED state). Motion while unlocked does not trigger an alarm. |
| Toggle `if state == LOCKED / elif state in (UNLOCKED, ALARM)` | Single card scan cycles between UNLOCKED and LOCKED, also acting as alarm reset. |

## Hardware & Safety Concept: Authentication-Gated Security Zones
Chaining an authentication gate (RFID) before activating a sensor zone (PIR) prevents the PIR from triggering false alarms when an authorized user is present. This is the basis for "disarm before entry" in commercial access control systems (e.g. ADT, Honeywell Vista panels).

## Try This! (Challenges)
1. **Multi-User UID List**: Change `AUTHORIZED_UID` from a single list to a list of lists and check if the scanned UID appears in the authorized list using `uid_list in AUTHORIZED_UIDS`.
2. **Entry Countdown**: After ACCESS GRANTED, display a 10-second countdown on the LCD before auto-locking again.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| RFID never responds | SPI wiring error | Confirm SCK→GP6, MOSI→GP7, MISO→GP8, SS→GP5, RST→GP9. |
| Alarm triggers immediately | PIR not stabilized | Add a 30-second warm-up delay after boot before enabling the PIR check. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [96 - Pico RFID Access Control LCD](96-pico-rfid-access-control-lcd.md)
- [109 - Pico RFID Access Control Full](109-pico-rfid-access-control-full.md)
- [85 - Pico Motion Security PIR LCD](85-pico-motion-security-pir-lcd.md)
