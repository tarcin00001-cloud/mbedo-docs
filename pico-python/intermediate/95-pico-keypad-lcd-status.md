# 95 - Pico Keypad LCD Status

Build a passcode entry terminal that scans a 4x4 matrix keypad, displays typed keys (or asterisks for privacy) on an I2C LCD screen, and verifies the passcode.

## Goal
Learn how to scan matrix keypads, implement multi-digit PIN buffer entry checks, and update/format status text and entry characters on an I2C character LCD in MicroPython.

## What You Will Build
A passcode entry terminal:
- **4x4 Keypad (GP2-GP9)**: Enters the passcode.
- **I2C 16x2 LCD (GP4, GP5)**: Displays asterisks `*` as you type, and shows verification status ("ACCESS GRANTED" or "ACCESS DENIED").
- **LED Green (GP15)**: Turns ON during authorized access.
- **LED Red (GP14)**: Flashes during unauthorized entries.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| Red, Green LEDs | `led` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 330 Ω Resistors × 2 | `resistor` | Optional in MbedO | Yes — one per LED |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| 4x4 Keypad | Rows R1-R4 | GP2-GP3, GP6-GP7 | Red / Orange | Output scanning lines |
| 4x4 Keypad | Columns C1-C4 | GP8-GP9, GP20-GP21 | Blue / Purple | Input reading lines |
| LED Red | Anode (+, longer leg) | GP14 | Red | Access Denied indicator |
| LED Green | Anode (+, longer leg) | GP15 | Green | Access Granted indicator |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the keypad rows to GP2-GP7, and columns to GP8-GP21. Connect the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime
from machine_lcd import I2cLcd

# Configure row pins as digital outputs
row_pins = [Pin(2, Pin.OUT, value=1), Pin(3, Pin.OUT, value=1), Pin(6, Pin.OUT, value=1), Pin(7, Pin.OUT, value=1)]

# Configure column pins as digital inputs with pull-up resistors
col_pins = [Pin(8, Pin.IN, Pin.PULL_UP), Pin(9, Pin.IN, Pin.PULL_UP), Pin(20, Pin.IN, Pin.PULL_UP), Pin(21, Pin.IN, Pin.PULL_UP)]

keys = [['1', '2', '3', 'A'], ['4', '5', '6', 'B'], ['7', '8', '9', 'C'], ['*', '0', '#', 'D']]

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

led_red   = Pin(14, Pin.OUT)
led_green = Pin(15, Pin.OUT)

# Target secret passcode and entry buffer
SECRET_PASSCODE = "1234"
entry_buffer = ""

# Ensure start state is dark
led_red.value(0)
led_green.value(0)

def update_display():
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("Enter Passcode:")
    lcd.move_to(0, 1)
    # Mask characters with asterisks
    mask_str = "*" * len(entry_buffer)
    lcd.putstr(mask_str)

def scan_keypad():
    pressed_key = None
    for r_idx in range(4):
        row_pins[r_idx].value(0)
        for c_idx in range(4):
            if col_pins[c_idx].value() == 0:
                pressed_key = keys[r_idx][c_idx]
                while col_pins[c_idx].value() == 0:
                    utime.sleep_ms(10)
        row_pins[r_idx].value(1)
    return pressed_key

update_display()
print("Keypad Lock armed. Enter PIN and press '#':")

while True:
    key = scan_keypad()
    
    if key is not None:
        if key == '#': # Enter command
            print("Verifying PIN...")
            lcd.clear()
            lcd.move_to(0, 0)
            lcd.putstr("Verifying...")
            utime.sleep_ms(500)
            
            if entry_buffer == SECRET_PASSCODE:
                print(">> ACCESS GRANTED.")
                led_green.value(1)
                lcd.clear()
                lcd.move_to(0, 0)
                lcd.putstr("ACCESS GRANTED")
                lcd.move_to(0, 1)
                lcd.putstr("WELCOME")
                utime.sleep(3) # Hold open for 3 seconds
                led_green.value(0)
            else:
                print(">> ACCESS DENIED!")
                led_red.value(1)
                lcd.clear()
                lcd.move_to(0, 0)
                lcd.putstr("ACCESS DENIED")
                lcd.move_to(0, 1)
                lcd.putstr("INVALID PIN")
                
                # Flash red LED warning
                for _ in range(4):
                    led_red.value(1)
                    utime.sleep_ms(100)
                    led_red.value(0)
                    utime.sleep_ms(100)
                    
            entry_buffer = "" # Clear buffer
            update_display()
        elif key == '*': # Clear command
            entry_buffer = ""
            update_display()
            print("Buffer cleared.")
        else:
            entry_buffer += key
            update_display()
            
    utime.sleep_ms(20)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **4x4 Keypad**, **two LEDs**, and **I2C LCD** onto the canvas.
2. Connect Keypad to **GP2-GP9**, LEDs to **GP14/GP15**, and LCD to **GP4/GP5**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click Keypad keys `1`, `2`, `3`, `4`, and `#` and observe the LCD.

## Expected Output
```
Keypad Lock armed. Enter PIN and press '#':
Verifying PIN...
>> ACCESS GRANTED.
```
(On screen: `****` showing on the second line as you type `1234`, changing to "ACCESS GRANTED" when verified.)

## Expected Canvas Behavior
- The green LED lights up and the LCD screen updates to show "ACCESS GRANTED" when the correct passcode is entered on the keypad component.
- Entering an incorrect passcode flashes the red LED and displays "ACCESS DENIED / INVALID PIN".

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `mask_str = "*" * len(entry_buffer)` | Creates a string of asterisks matching the length of the typed passcode buffer to mask the PIN. |
| `entry_buffer == SECRET_PASSCODE` | Compares the accumulated key entries with the target passcode. |

## Hardware & Safety Concept: Entry Masking and Privacy
Passcode entry systems mask typed digits with asterisks or dots. This prevents bystanders from reading the PIN over the user's shoulder (shoulder surfing). Privacy masking is a basic requirement in ATM design and security lock keypads.

## Try This! (Challenges)
1. **Indicator Buzzer**: Connect a buzzer on GP13 and sound a high double beep on success, and a low warning tone on failure.
2. **Backspace Key**: Configure the `*` key to act as a backspace, deleting only the last character typed in the buffer.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Key presses are not registered | Keypad pins mapped incorrectly | Double-check that keypad row and column pins match the GP pin assignments in your code. |
| LCD does not update on key press | Update display routine omitted | Ensure the `update_display()` helper is called after modifying the `entry_buffer`. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [64 - Pico Keypad Serial](64-pico-keypad-serial.md)
- [89 - Pico Keypad Lock](89-pico-keypad-lock.md)
