# 126 - Pico MPU-6050 Tilt Security Vault

Build a tilt-triggered security vault that requires RFID authentication to unlock, monitors for tampering using an MPU-6050 accelerometer, and triggers an alarm if the device is tilted while locked.

## Goal
Learn how to chain RFID authentication with accelerometer tamper detection, use I2C to read the MPU-6050 raw acceleration values, compare them against tilt thresholds, and manage a multi-state alarm system with servo lock and LCD status in MicroPython.

## What You Will Build
A tilt-protected security vault:
- **MFRC522 RFID (SPI: GP5-GP9)**: Authorizes access. Card swipe toggles lock/unlock.
- **MPU-6050 IMU (GP4, GP5 I2C)**: Monitors orientation continuously. Tilt above threshold while locked = alarm.
- **Servo Motor (GP11)**: Controls the vault lock (0° = locked, 90° = open).
- **Red LED (GP13)**: Alarm state indicator.
- **Buzzer (GP15)**: Tamper alarm siren.
- **I2C 16x2 LCD (GP2, GP3)**: Displays lock state and tilt values.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| MPU-6050 IMU Sensor | `mpu6050` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |
| Red LED | `led` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 | SDA (SS) | GP5 | White | SPI Chip Select |
| MFRC522 | SCK | GP6 | Blue | SPI Clock |
| MFRC522 | MOSI | GP7 | Orange | SPI Master Out |
| MFRC522 | MISO | GP8 | Yellow | SPI Master In |
| MFRC522 | RST | GP9 | Purple | Hardware Reset |
| MFRC522 | VCC / GND | 3.3V / GND | Red / Black | Module power |
| MPU-6050 | SDA | GP4 | Orange | I2C Bus 0 data |
| MPU-6050 | SCL | GP5 | Blue | I2C Bus 0 clock (NOTE: shared with RFID SDA — use separate SPI/I2C bus mapping) |
| MPU-6050 | VCC / GND | 3.3V / GND | Red / Black | Module power |
| I2C LCD | SDA / SCL | GP2 / GP3 | Orange / Blue | I2C Bus 1 data/clock |
| Servo Motor | Control (PWM) | GP11 | Orange | Servo PWM signal |
| Servo Motor | Power (+) | 5V (VBUS) | Red | Servo power |
| Red LED | Anode (+, via 330 Ω) | GP13 | Red | Alarm indicator |
| Red LED | Cathode (−) | GND | Black | Ground return |
| Active Buzzer | VCC (+) | GP15 | Red | Tamper alarm output |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The MPU-6050 and RFID coexist on separate buses: RFID uses SPI (GP5-GP9) and MPU-6050 uses I2C Bus 0 (GP4/GP5). The LCD uses I2C Bus 1 (GP2/GP3) to avoid conflicts. All grounds are shared.

## Code
```python
from machine import Pin, PWM, SPI, I2C
import utime
import struct
from mfrc522 import MFRC522
from machine_lcd import I2cLcd

# SPI for MFRC522
spi  = SPI(0, sck=Pin(6), mosi=Pin(7), miso=Pin(8))
rfid = MFRC522(spi=spi, gpioRst=9, gpioCs=5)

# I2C Bus 0 for MPU-6050
i2c0 = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)

# I2C Bus 1 for LCD
i2c1 = I2C(1, sda=Pin(2), scl=Pin(3), freq=400000)
lcd  = I2cLcd(i2c1, 0x27, 2, 16)

# Servo on GP11
servo = PWM(Pin(11))
servo.freq(50)

alarm_led = Pin(13, Pin.OUT)
buzzer    = Pin(15, Pin.OUT)

# MPU-6050 I2C address and registers
MPU_ADDR  = 0x68
REG_PWR   = 0x6B
REG_ACCX  = 0x3B

# Wake MPU-6050 up (clear sleep bit)
i2c0.writeto_mem(MPU_ADDR, REG_PWR, b'\x00')

# Authorized UID
AUTHORIZED_UID = [0xDE, 0xAD, 0xBE, 0xEF]

# Tilt threshold (raw accelerometer units, 16384 = 1g for ±2g range)
TILT_THRESHOLD = 5000

def set_servo(angle):
    duty = 1638 + int(angle * (8192 - 1638) / 180)
    servo.duty_u16(duty)

def read_accel():
    data = i2c0.readfrom_mem(MPU_ADDR, REG_ACCX, 6)
    ax   = struct.unpack('>h', data[0:2])[0]
    ay   = struct.unpack('>h', data[2:4])[0]
    az   = struct.unpack('>h', data[4:6])[0]
    return ax, ay, az

# System states
LOCKED   = "LOCKED"
UNLOCKED = "UNLOCKED"
ALARM    = "ALARM"

state = LOCKED
set_servo(0)
alarm_led.value(0)
buzzer.value(0)

lcd.clear()
lcd.putstr("Tilt Vault Ready")
utime.sleep(1)

print("Tilt security vault active.")

while True:
    # Read accelerometer
    ax, ay, az = read_accel()
    tilt = abs(ax) > TILT_THRESHOLD or abs(ay) > TILT_THRESHOLD
    
    # Trigger tamper alarm if tilted while locked
    if state == LOCKED and tilt:
        state = ALARM
        alarm_led.value(1)
        print(">> TAMPER DETECTED. Ax:{} Ay:{}".format(ax, ay))
        
    # RFID scan
    rfid.init()
    (stat, _) = rfid.request(rfid.REQIDL)
    if stat == rfid.OK:
        (stat, raw_uid) = rfid.anticoll()
        if stat == rfid.OK:
            uid = list(raw_uid[:4])
            if uid == AUTHORIZED_UID:
                if state in (LOCKED, ALARM):
                    state = UNLOCKED
                    set_servo(90)
                    alarm_led.value(0)
                    buzzer.value(0)
                    print(">> ACCESS GRANTED. Vault open.")
                else:
                    state = LOCKED
                    set_servo(0)
                    print(">> VAULT LOCKED.")
            else:
                print(">> DENIED:", [hex(b) for b in uid])
                
    # LCD Update
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("Vault: " + state[:8])
    lcd.move_to(0, 1)
    lcd.putstr("Ax:{:5d} Ay:{:5d}".format(ax, ay))
    
    # Alarm pulsing
    if state == ALARM:
        buzzer.value(1)
        utime.sleep_ms(250)
        buzzer.value(0)
        utime.sleep_ms(250)
    else:
        utime.sleep_ms(150)
```

## What to Click in MbedO
1. Drag the **Pico**, **MFRC522**, **MPU-6050**, **Servo**, **Red LED**, **Buzzer**, and **I2C LCD** onto the canvas.
2. Wire according to the wiring table. All grounds shared.
3. Update `AUTHORIZED_UID` to match your card. Click **Run**.
4. In MbedO, slide the MPU-6050 Ac_x or Ac_y slider above 5000 while the vault is LOCKED to trigger the tamper alarm. Scan the authorized card to reset.

## Expected Output
```
Tilt security vault active.
>> TAMPER DETECTED. Ax:6200 Ay:800
>> ACCESS GRANTED. Vault open.
>> VAULT LOCKED.
```

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `struct.unpack('>h', data[0:2])[0]` | Unpacks a big-endian signed 16-bit integer from two MPU-6050 register bytes. |
| `state == LOCKED and tilt` | Tamper alarm only fires when the vault is in the locked state. Tilt while open is ignored. |

## Hardware & Safety Concept: Accelerometer-Based Tamper Detection
High-security enclosures use accelerometer chips as **motion/tilt tamper switches**. If the enclosure is tilted or moved unexpectedly, the alarm triggers before the physical lock can be attacked. Some HSMs (Hardware Security Modules) are designed to erase their cryptographic keys if tilt exceeds a threshold.

## Try This! (Challenges)
1. **Z-Axis Freefall**: Add a freefall detection condition (`abs(az) < 2000`) that triggers an extreme alarm — e.g. the vault was dropped.
2. **Alarm Lockout**: After 3 unauthorized scans, lock the RFID reader for 60 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| MPU-6050 returns all zeros | Wake-up command not sent | Ensure `i2c0.writeto_mem(MPU_ADDR, REG_PWR, b'\x00')` is called before reading. |
| LCD garbled | I2C bus conflict | Confirm LCD is on I2C Bus 1 (GP2/GP3) and MPU-6050 is on I2C Bus 0 (GP4/GP5). |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [96 - Pico RFID Access Control LCD](../intermediate/96-pico-rfid-access-control-lcd.md)
- [121 - Pico RFID PIR Smart Lock LCD](121-pico-rfid-pir-smart-lock-lcd.md)
