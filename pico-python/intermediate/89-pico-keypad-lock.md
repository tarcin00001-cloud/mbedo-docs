# 89 - Pico Keypad Lock

Build a password security lock box that scans a 4x4 matrix keypad, unlocks a servo door latch when the correct PIN code is entered, and displays safety warnings on indicators.

## Goal
Learn how to scan matrix keypads, implement multi-digit PIN buffer entry checks, control servo steering, and play warning chimes on passive buzzers in MicroPython.

## What You Will Build
An automated keypad lock box:
- **4x4 Keypad (GP2-GP9)**: Enters the passcode.
- **Servo Motor (GP10)**: Steers to 90 degrees (unlocked) for 3 seconds on correct PIN entry, then returns to 0 degrees (locked).
- **LED Green (GP15)**: Turns ON during authorized access.
- **LED Red (GP14)**: Flashes if an incorrect PIN code is entered.
- **Passive Buzzer (GP13)**: Emits a high confirmation beep on success, and a low warning tone for failure.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes (standard SG90) |
| Red, Green LEDs | `led` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes (passive required) |
| 330 Ω Resistors × 2 | `resistor` | Optional in MbedO | Yes — one per LED |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| 4x4 Keypad | Rows R1-R4 | GP2-GP3, GP6-GP7 | Red / Orange | Output scanning lines |
| 4x4 Keypad | Columns C1-C4 | GP8-GP9, GP20-GP21 | Blue / Purple | Input reading lines |
| Servo Motor | Control (PWM) | GP10 | Orange | PWM servo signal |
| Servo Motor | Power (+) | 5V (VBUS) | Red | Servo power supply (5V) |
| LED Red | Anode (+, longer leg) | GP14 | Red | Access Denied indicator |
| LED Green | Anode (+, longer leg) | GP15 | Green | Access Granted indicator |
| Passive Buzzer | Signal (+) | GP13 | Grey | Warning chime signal (PWM) |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the keypad rows to GP2-GP7, and columns to GP8-GP21. Connect the servo control pin to GP10, and power to 5V VBUS. Connect the LEDs and passive buzzer to their respective pins. All grounds are shared.

## Code
```python
from machine import Pin, PWM
import utime

# Configure row pins as digital outputs
row_pins = [Pin(2, Pin.OUT, value=1), Pin(3, Pin.OUT, value=1), Pin(6, Pin.OUT, value=1), Pin(7, Pin.OUT, value=1)]

# Configure column pins as digital inputs with pull-up resistors
col_pins = [Pin(8, Pin.IN, Pin.PULL_UP), Pin(9, Pin.IN, Pin.PULL_UP), Pin(20, Pin.IN, Pin.PULL_UP), Pin(21, Pin.IN, Pin.PULL_UP)]

keys = [['1', '2', '3', 'A'], ['4', '5', '6', 'B'], ['7', '8', '9', 'C'], ['*', '0', '#', 'D']]

# Set up Servo on GP10 (50 Hz PWM)
servo = PWM(Pin(10))
servo.freq(50)

# Set up Passive Buzzer on GP13
buzzer = PWM(Pin(13))

led_red   = Pin(14, Pin.OUT)
led_green = Pin(15, Pin.OUT)

# Target secret passcode and entry buffer
SECRET_PASSCODE = "1234"
entry_buffer = ""

# Servo duty cycle limits
MIN_DUTY = 1638 # Locked (0 deg)
MAX_DUTY = 4915 # Unlocked (90 deg)

# Ensure start state is locked and silent
servo.duty_u16(MIN_DUTY)
led_red.value(0)
led_green.value(0)
buzzer.duty_u16(0)

def set_lock(open_state):
    duty = MAX_DUTY if open_state else MIN_DUTY
    servo.duty_u16(duty)

def sound_chime(success):
    if success:
        # Success chime (High pitched double beep)
        for freq in [1000, 1500]:
            buzzer.freq(freq)
            buzzer.duty_u16(32768)
            utime.sleep_ms(80)
        buzzer.duty_u16(0)
    else:
        # Failure chime (Low warning tone)
        buzzer.freq(220)
        buzzer.duty_u16(32768)
        utime.sleep_ms(400)
        buzzer.duty_u16(0)

def scan_keypad():
    pressed_key = None
    for r_idx in range(4):
        row_pins[r_idx].value(0)
        for c_idx in range(4):
            if col_pins[c_idx].value() == 0:
                pressed_key = keys[r_idx][c_idx]
                # Key press feedback tone
                buzzer.freq(800)
                buzzer.duty_u16(16384)
                utime.sleep_ms(50)
                buzzer.duty_u16(0)
                
                while col_pins[c_idx].value() == 0:
                    utime.sleep_ms(10)
        row_pins[r_idx].value(1)
    return pressed_key

print("Keypad Lock armed. Enter PIN and press '#':")

while True:
    key = scan_keypad()
    
    if key is not None:
        if key == '#': # Enter command
            print("Verifying PIN:", entry_buffer)
            
            if entry_buffer == SECRET_PASSCODE:
                print(">> ACCESS GRANTED. Unlocking...")
                led_green.value(1)
                sound_chime(True)
                set_lock(True) # Unlock door
                
                utime.sleep(3) # Hold open for 3 seconds
                
                set_lock(False) # Lock door
                led_green.value(0)
                print(">> Door locked.")
            else:
                print(">> ACCESS DENIED! Invalid PIN.")
                led_red.value(1)
                sound_chime(False)
                
                # Flash red LED warning
                for _ in range(4):
                    led_red.value(1)
                    utime.sleep_ms(100)
                    led_red.value(0)
                    utime.sleep_ms(100)
                    
            entry_buffer = "" # Clear buffer
        elif key == '*': # Clear command
            entry_buffer = ""
            print("Buffer cleared.")
        else:
            entry_buffer += key
            print("Key input: *")
            
    utime.sleep_ms(20)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **4x4 Keypad**, **Servo Motor**, **two LEDs**, and **Passive Buzzer** onto the canvas.
2. Connect Keypad to **GP2-GP9**, Servo to **GP10**, LEDs to **GP14/GP15**, and Buzzer to **GP13**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click Keypad keys `1`, `2`, `3`, `4`, and `#` to unlock the servo.

## Expected Output
```
Keypad Lock armed. Enter PIN and press '#':
Key input: *
Key input: *
Key input: *
Key input: *
Verifying PIN: 1234
>> ACCESS GRANTED. Unlocking...
>> Door locked.
```

## Expected Canvas Behavior
- The green LED lights up and the servo motor rotates to 90 degrees when the correct passcode is entered on the keypad component.
- Entering an incorrect passcode flashes the red LED and sounds the warning chime.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `entry_buffer == SECRET_PASSCODE` | Compares the accumulated key entries with the target passcode. |
| `set_lock(True)` | Unlocks the door latch by steering the servo to 90 degrees. |

## Hardware & Safety Concept: Temporary Lockout Loops
To protect passcode locks from automated testing attacks (brute forcing), security firmware tracks consecutive failed attempts. If a user enters an incorrect PIN three times in a row, the controller disables all keypad input for a **temporary lockout period** (e.g. 5 minutes) and sounds an alarm, preventing further entry attempts.

## Try This! (Challenges)
1. **Passcode Lockout**: Add a failed attempt counter. Lock out keypad input for 10 seconds if an incorrect PIN is entered three times consecutively.
2. **Master Pin**: Add a secondary, hardcoded override PIN code that always grants access.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Key presses are not registered | Keypad pins mapped incorrectly | Double-check that keypad row and column pins match the GP pin assignments in your code. |
| Servo does not rotate | Power supply overload | Connect the servo's power (+) pin to the 5V VBUS pin, not 3.3V. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [59 - Pico Servo Knob](59-pico-servo-knob.md)
- [64 - Pico Keypad Serial](64-pico-keypad-serial.md)
- [88 - Pico RFID Access Control](88-pico-rfid-access-control.md)
