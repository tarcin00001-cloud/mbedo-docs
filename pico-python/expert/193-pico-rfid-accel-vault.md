# 193 - Pico RFID Vault with Acceleration Safeguards

Build a high-security automated vault door that opens using an RFID card, controls a lock servo, and locks out access immediately if an MPU-6050 accelerometer detects physical tampering or shaking.

## Goal
Learn how to interface an MFRC522 RFID reader over SPI, read an MPU-6050 accelerometer over I2C, coordinate multi-device safety states, and implement security lockout overrides in MicroPython.

## What You Will Build
A tamper-proof electronic vault:
- **MFRC522 RFID Reader (SPI0: GP16-GP20)**: Scans keycards to unlock the vault.
- **MPU-6050 Accelerometer (I2C0: GP4, GP5)**: Detects physical tilting or shaking of the vault box.
- **Locking Servo (GP10)**: Acts as the physical vault lock (0° locked, 90° unlocked).
- **Alarm Buzzer (GP11)**: Sounds yelp alerts during tampering.
- **Reset Button (GP13)**: Dismisses the tamper alarm and re-arms the vault.
- **16x2 I2C LCD (GP4, GP5 shared)**: Displays active status, keycard results, and tamper warnings.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `rfid` | Yes | Yes |
| MPU-6050 Accelerometer | `gyroscope` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 | SCK / MOSI / MISO | GP18 / GP19 / GP16 | Blue/Orange/Yellow | SPI0 bus |
| MFRC522 | SDA (CS) / RST | GP17 / GP20 | White / Purple | SPI Chip Select & Reset |
| MFRC522 | VCC / GND | 3.3V / GND | Red / Black | Module power |
| MPU-6050 | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| MPU-6050 | VCC / GND | 3.3V / GND | Red / Black | Sensor power |
| Locking Servo | Control (PWM) | GP10 | Orange | Lock servo signal |
| Locking Servo | VCC / GND | 5V / GND | Red / Black | Servo power lines |
| Buzzer | VCC (+) | GP11 | Blue | Tamper alarm output |
| Reset Button | Terminal 1 | GP13 | White | Alarm reset switch |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The MFRC522 uses SPI0 on GP16-GP20. The MPU-6050 and I2C LCD share I2C Bus 0 on GP4/GP5. The Locking Servo is on GP10, Buzzer on GP11, and Reset Button on GP13.

## Code
```python
from machine import Pin, PWM, I2C, SPI
import utime, machine
from machine_lcd import I2cLcd
from mfrc522 import MFRC522

# Servo Setup
lock_servo = PWM(Pin(10)); lock_servo.freq(50)
buzzer = Pin(11, Pin.OUT); buzzer.value(0)
btn_reset = Pin(13, Pin.IN, Pin.PULL_UP)

# I2C Bus 0: LCD & MPU-6050
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# MFRC522 SPI0 Setup
spi = SPI(0, sck=Pin(18), mosi=Pin(19), miso=Pin(16))
reader = MFRC522(spi, Pin(17), Pin(20))  # sda=GP17, rst=GP20

# Valid Card UID pattern (adjust for your card)
VALID_UID = [0x12, 0x34, 0x56, 0x78]

def set_lock(angle):
    duty = 1638 + int(angle * (8192 - 1638) / 180)
    lock_servo.duty_u16(duty)

def get_mpu_accel():
    """Reads acceleration data from MPU-6050 registers."""
    try:
        # MPU-6050 address is 0x68. Read 6 bytes starting at ACCEL_XOUT_H (0x3B)
        data = i2c.readfrom_mem(0x68, 0x3B, 6)
        # Convert bytes to 16-bit signed integers
        x = (data[0] << 8) | data[1]
        y = (data[2] << 8) | data[3]
        z = (data[4] << 8) | data[5]
        if x > 32767: x -= 65536
        if y > 32767: y -= 65536
        if z > 32767: z -= 65536
        return x, y, z
    except OSError:
        return 0, 0, 16384  # Default gravity on Z if read fails

# Vault State
TAMPER_LIMIT = 8000  # Acceleration delta threshold
tamper_alarm = False
vault_locked = True

set_lock(0)  # Close lock
lcd.clear(); lcd.putstr("Security Vault\nArmed & Secure")
utime.sleep(1.5)

# Establish baseline gravity values
base_x, base_y, base_z = get_mpu_accel()
print("System armed. Baseline: X:{} Y:{} Z:{}".format(base_x, base_y, base_z))

while True:
    now_x, now_y, now_z = get_mpu_accel()
    
    # Calculate acceleration change (motion delta)
    dx = abs(now_x - base_x)
    dy = abs(now_y - base_y)
    
    # 1. Tamper detection: if box is shaken or tilted
    if (dx > TAMPER_LIMIT or dy > TAMPER_LIMIT) and not tamper_alarm:
        tamper_alarm = True
        vault_locked = True
        set_lock(0)  # Force lock closed
        print("!! SECURITY BREACH: TAMPER DETECTED !!")
        
    # 2. Handle alarm state
    if tamper_alarm:
        # Yelp siren
        buzzer.value(1)
        utime.sleep_ms(80)
        buzzer.value(0)
        utime.sleep_ms(80)
        
        lcd.clear()
        lcd.putstr("!! TAMPER !!\nVault Locked")
        
        # Check for reset button click (GP13)
        if btn_reset.value() == 0:
            tamper_alarm = False
            # Re-establish baseline
            base_x, base_y, base_z = get_mpu_accel()
            print("Alarm reset. Re-armed.")
            lcd.clear(); lcd.putstr("System Reset\nArmed & Secure")
            utime.sleep(1.5)
            
    # 3. Normal operating state
    else:
        # Check for card scan
        (stat, tag_type) = reader.request(reader.REQIDL)
        if stat == reader.OK:
            (stat, uid) = reader.anticoll()
            if stat == reader.OK:
                print("Card Scanned. UID:", [hex(b) for b in uid])
                
                # UID match check (checks first 4 bytes)
                if uid[0:4] == VALID_UID:
                    lcd.clear()
                    lcd.putstr("Access Granted\nDoors Opening...")
                    set_lock(90)  # Unlock vault
                    buzzer.value(1); utime.sleep_ms(150); buzzer.value(0)
                    
                    utime.sleep(5.0)  # Keep open for 5 seconds
                    
                    set_lock(0)   # Relock
                    lcd.clear(); lcd.putstr("Doors Closed\nVault Locked")
                    utime.sleep(1.0)
                else:
                    lcd.clear()
                    lcd.putstr("Access Denied\nInvalid Card")
                    # Low buzz error
                    for _ in range(3):
                        buzzer.value(1); utime.sleep_ms(100); buzzer.value(0); utime.sleep_ms(50)
                    utime.sleep(1.0)
                    
        # Update normal display
        lcd.clear()
        lcd.putstr("Vault Locked\nScan Card...")
        utime.sleep_ms(200)
```

## What to Click in MbedO
1. Drag **Pico**, **MFRC522**, **MPU-6050**, **Servo**, **I2C LCD**, **Buzzer**, and **Push Button** onto the canvas.
2. Connect MFRC522 to **GP16-GP20**, MPU-6050 + LCD to **GP4/GP5**, Servo to **GP10**, Buzzer to **GP11**, and Button to **GP13**.
3. Paste code, select the interpreted mode, and click **Run**.
4. In MbedO, click the RFID sensor to simulate a valid card swipe. Observe the servo unlock.
5. Shake the MPU-6050 (move its tilt slider). Observe the buzzer yelping. Swiping the card will now be locked out. Click the reset button to disarm.

## Expected Output
Terminal:
```
System armed. Baseline: X:120 Y:-80 Z:16400
Card Scanned. UID: ['0x12', '0x34', '0x56', '0x78']
!! SECURITY BREACH: TAMPER DETECTED !!
Alarm reset. Re-armed.
```

## Expected Canvas Behavior
* Boot: LCD reads `Armed & Secure`. Servo is at 0°.
* Swipe Valid Card: Servo rotates to 90° for 5 seconds. LCD reads `Access Granted`.
* Move MPU-6050: Buzzer yelps. LCD reads `!! TAMPER !!`. Cards are ignored.
* Press Button (GP13): Buzzer stops, system re-arms.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `i2c.readfrom_mem(0x68, 0x3B, 6)` | Reads 6 bytes of acceleration raw registers from the MPU-6050. |
| `reader.anticoll()` | Runs the collision prevention loop to retrieve the UID of the scanned RFID card. |

## Hardware & Safety Concept: Physical Security Locks
Electronic locks are prone to physical attacks, such as shaking or dropping the vault box to force the solenoid mechanism to slip open. Integrating a multi-axis accelerometer detects any physical impact or drop immediately. Safe industrial security systems use a **lockout state**: if tampering is detected, the controller cuts power to the door actuators and locks the mechanism down, requiring manual operator override.

## Try This! (Challenges)
1. **Password Override**: If the vault is in the TAMPER state, allow entering a PIN sequence over UART to reset it without pressing the button.
2. **Door Contact Switch**: Add a limit switch on GP12 to trigger the alarm if the door remains open for more than 10 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Tamper alarm triggers immediately on boot | Baseline capture issue | Ensure the MPU-6050 is sitting completely still when the program starts so the baseline gravity readings are accurate. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [126 - Pico MPU6050 Tilt Security Vault](126-pico-mpu6050-tilt-security-vault.md)
- [137 - Pico Keypad RFID Dual Auth System](137-pico-keypad-rfid-dual-auth-system.md)
