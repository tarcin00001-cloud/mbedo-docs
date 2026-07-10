# 199 - Pico Keypad Password Lockout

Build a secure digital door lock that processes a 4-digit PIN using a 4x4 matrix keypad, controls a lock servo, and locks out inputs for 60 seconds after three wrong attempts.

## Goal
Learn how to scan a 4x4 matrix keypad, compare password strings, implement lockout delay timers, and actuate status indicator LEDs and servo locks in MicroPython.

## What You Will Build
A security pin-pad lock:
- **4x4 Matrix Keypad (GP10 to GP17)**: Enables numeric PIN entry.
- **Locking Servo (GP18)**: Operates the door lock mechanism.
- **Alarm Buzzer (GP19)**: Sounds sirens during lockout breaches.
- **Green LED (GP20)**: Illuminates on successful access.
- **Red LED (GP21)**: Illuminates during incorrect entry and lockouts.
- **16x2 I2C LCD (GP4, GP5)**: Displays entry feedback, attempts left, and lockout countdowns.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes (membrane or button matrix) |
| Servo Motor | `servo` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Green & Red LEDs | `led` | Yes (two LEDs) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| 4x4 Keypad | Rows 1–4 | GP10 to GP13 | Yellow wires | Keypad row scan lines |
| 4x4 Keypad | Cols 1–4 | GP14 to GP17 | Green wires | Keypad column inputs |
| Locking Servo | Control (PWM) | GP18 | Orange | Lock servo signal line |
| Locking Servo | VCC / GND | 5V / GND | Red / Black | Servo power lines |
| Buzzer | VCC (+) | GP19 | Blue | Alarm siren pin |
| Green LED | Anode (via 330 Ω) | GP20 | Green | Correct PIN indicator |
| Red LED | Anode (via 330 Ω) | GP21 | Red | Wrong PIN / Lockout indicator |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Keypad rows connect to GP10-GP13 (outputs), and columns connect to GP14-GP17 (inputs with pull-ups). The Servo signal is GP18, Buzzer is GP19, and LEDs are on GP20/GP21.

## Code
```python
from machine import Pin, PWM, I2C
import utime
from machine_lcd import I2cLcd

# Keypad GPIO Rows and Cols
row_pins = [Pin(10, Pin.OUT), Pin(11, Pin.OUT), Pin(12, Pin.OUT), Pin(13, Pin.OUT)]
col_pins = [Pin(14, Pin.IN, Pin.PULL_UP), Pin(15, Pin.IN, Pin.PULL_UP),
            Pin(16, Pin.IN, Pin.PULL_UP), Pin(17, Pin.IN, Pin.PULL_UP)]

# Keypad Map
KEYMAP = [
    ['1', '2', '3', 'A'],
    ['4', '5', '6', 'B'],
    ['7', '8', '9', 'C'],
    ['*', '0', '#', 'D']
]

# Actuators & Indicators
lock_servo = PWM(Pin(18)); lock_servo.freq(50)
buzzer = Pin(19, Pin.OUT)
led_green = Pin(20, Pin.OUT)
led_red = Pin(21, Pin.OUT)

buzzer.value(0)
led_green.value(0)
led_red.value(0)

# Setup I2C LCD
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Security Parameters
CORRECT_PIN = "1234"
wrong_attempts = 0
MAX_ATTEMPTS = 3
lockout_duration = 30  # seconds

entered_pin = ""

def set_lock(angle):
    duty = 1638 + int(angle * (8192 - 1638) / 180)
    lock_servo.duty_u16(duty)

def scan_keypad():
    """Scans the 4x4 matrix keypad and returns the character pressed, or None."""
    for r_idx, r_pin in enumerate(row_pins):
        # Set current row LOW
        r_pin.value(0)
        
        for c_idx, c_pin in enumerate(col_pins):
            if c_pin.value() == 0:  # Pin pulled LOW (pressed)
                # Wait for release (debounce)
                while c_pin.value() == 0:
                    utime.sleep_ms(10)
                # Restore row to HIGH
                r_pin.value(1)
                return KEYMAP[r_idx][c_idx]
                
        # Restore row to HIGH
        r_pin.value(1)
    return None

set_lock(0)  # Close lock
lcd.clear(); lcd.putstr("Secure Lock\nArmed & Active")
utime.sleep(1.5)

print("Keypad secure lock active.")

while True:
    # 1. Handle Active Lockout State
    if wrong_attempts >= MAX_ATTEMPTS:
        print(">> SYSTEM LOCKED OUT due to consecutive failures!")
        led_red.value(1)
        
        for seconds_left in range(lockout_duration, 0, -1):
            lcd.clear()
            lcd.putstr("!! LOCKOUT !!\nWait: {}s".format(seconds_left))
            
            # Pulse buzzer alarm
            buzzer.value(1)
            utime.sleep_ms(100)
            buzzer.value(0)
            utime.sleep_ms(900)  # Total 1 second delay loop
            
        # Lockout period expired: reset stats
        led_red.value(0)
        wrong_attempts = 0
        entered_pin = ""
        print("Lockout cleared. System re-armed.")
        lcd.clear(); lcd.putstr("System Re-armed\nEnter PIN:")
        utime.sleep(1.0)

    # 2. Normal Pin-entry scan
    lcd.clear()
    lcd.putstr("Enter PIN: {}\n".format("*" * len(entered_pin)))
    lcd.putstr("Attempts left: {}".format(MAX_ATTEMPTS - wrong_attempts))
    
    # Read key
    key = scan_keypad()
    if key is not None:
        print("Key clicked:", key)
        # Beep feedback on click
        buzzer.value(1); utime.sleep_ms(50); buzzer.value(0)
        
        if key == '#':  # Submit PIN
            if entered_pin == CORRECT_PIN:
                # Access granted
                print(">> ACCESS GRANTED")
                led_green.value(1)
                lcd.clear(); lcd.putstr("Access Granted\nLock Opening...")
                set_lock(90)  # Open lock
                
                # Success double chirp
                buzzer.value(1); utime.sleep_ms(100); buzzer.value(0)
                utime.sleep_ms(100)
                buzzer.value(1); utime.sleep_ms(100); buzzer.value(0)
                
                utime.sleep(5.0)  # Hold open
                
                # Relock
                set_lock(0)
                led_green.value(0)
                entered_pin = ""
                wrong_attempts = 0
            else:
                # Access denied
                wrong_attempts += 1
                print(">> ACCESS DENIED. Fail count:", wrong_attempts)
                led_red.value(1)
                lcd.clear(); lcd.putstr("Access Denied\nInvalid PIN!")
                
                # Low beep error
                for _ in range(3):
                    buzzer.value(1); utime.sleep_ms(80); buzzer.value(0); utime.sleep_ms(50)
                led_red.value(0)
                entered_pin = ""
                utime.sleep(1.0)
                
        elif key == '*':  # Clear entry
            entered_pin = ""
        else:  # Append key
            if len(entered_pin) < 4:
                entered_pin += key
                
    utime.sleep_ms(50)
```

## What to Click in MbedO
1. Drag **Pico**, **Keypad**, **Servo**, **Buzzer**, **two LEDs**, and **I2C LCD** onto the canvas.
2. Connect Keypad to **GP10-GP17**, Servo to **GP18**, Buzzer to **GP19**, LEDs to **GP20/GP21**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. In MbedO, click the keypad buttons `1`, `2`, `3`, `4`, then click `#`. Verify that the green LED turns ON and the servo rotates.
5. Enter three wrong codes consecutively. Verify that the system enters lockout mode, displays a countdown on the LCD, and sounds the alarm.

## Expected Output
Terminal:
```
Keypad secure lock active.
Key clicked: 1
Key clicked: 2
Key clicked: 3
Key clicked: 4
Key clicked: #
>> ACCESS GRANTED
>> ACCESS DENIED. Fail count: 1
>> SYSTEM LOCKED OUT due to consecutive failures!
```

## Expected Canvas Behavior
* Normal state: LCD reads `Enter PIN:`. Green and Red LEDs are OFF. Servo is at 0°.
* Correct input (`1234#`): Green LED turns ON, Servo turns to 90° for 5 seconds.
* Incorrect input (three times): Red LED turns ON, buzzer sounds pulsing alarm, LCD counts down from 30 seconds. Keys are ignored.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Pin(14, Pin.IN, Pin.PULL_UP)` | Sets column pins as inputs with internal pull-ups. They read HIGH unless a row pin goes LOW and the key is clicked. |
| `entered_pin == CORRECT_PIN` | Compares the entered keypad string to the master PIN stored in memory. |

## Hardware & Safety Concept: Brute Force Prevention Lockouts
Security keypads are susceptible to brute-force attacks (automatically entering all possible combinations). To prevent this, commercial alarm keypads restrict the number of consecutive failed attempts. If the limit is breached, the controller initiates a lockout delay during which all inputs are disabled. In high-security systems, a lockout also transmits a notification to a monitoring station to report a tampering attempt.

## Try This! (Challenges)
1. **Dynamic Master Password**: Add a routine that allows the user to change the master PIN by pressing `'C'`, typing the old PIN, then typing a new 4-digit PIN.
2. **Duress code alert**: Add a special code (e.g. `9999`). If entered, open the servo lock but pulse the buzzer continuously to simulate a silent duress alarm.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Key clicks are detected repeatedly | Debounce delay too short | Ensure the scanning loop checks that the key is fully released (`while c_pin.value() == 0: pass`) before returning the character. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [108 - Pico 4x4 Keypad Password Lock Relay](../../advanced/108-pico-4x4-keypad-password-lock-relay.md)
- [137 - Pico Keypad RFID Dual Auth System](../advanced/137-pico-keypad-rfid-dual-auth-system.md)
