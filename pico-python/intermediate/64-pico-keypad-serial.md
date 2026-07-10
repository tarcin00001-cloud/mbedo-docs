# 64 - Pico Keypad Serial

Scan a 4x4 matrix keypad for key presses and print the values to the serial monitor.

## Goal
Learn how to implement a row-scanning multiplexing algorithm to read multiple keys using a limited number of GPIO pins in MicroPython.

## What You Will Build
A digital entry logger:
- **4x4 Matrix Keypad (GP2-GP9)**: Scans for key presses.
- **Serial Output**: Prints the value of the pressed key (0-9, A-D, *, #) to the terminal immediately.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| 4x4 Keypad | Row 1 (R1) | GP2 | Red | Output scanning line 1 |
| 4x4 Keypad | Row 2 (R2) | GP3 | Orange | Output scanning line 2 |
| 4x4 Keypad | Row 3 (R3) | GP6 | Yellow | Output scanning line 3 |
| 4x4 Keypad | Row 4 (R4) | GP7 | Green | Output scanning line 4 |
| 4x4 Keypad | Column 1 (C1) | GP8 | Blue | Input reading line 1 |
| 4x4 Keypad | Column 2 (C2) | GP9 | Purple | Input reading line 2 |
| 4x4 Keypad | Column 3 (C3) | GP20 | Grey | Input reading line 3 |
| 4x4 Keypad | Column 4 (C4) | GP21 | White | Input reading line 4 |

> **Wiring tip:** Connect the four row pins of the keypad to GP2-GP3 and GP6-GP7, and the four column pins to GP8-GP9 and GP20-GP21. The column pins are configured as inputs with internal pull-up resistors.

## Code
```python
from machine import Pin
import utime

# Configure row pins as digital outputs (start HIGH)
row_pins = [
    Pin(2, Pin.OUT, value=1),
    Pin(3, Pin.OUT, value=1),
    Pin(6, Pin.OUT, value=1),
    Pin(7, Pin.OUT, value=1)
]

# Configure column pins as digital inputs with pull-up resistors
col_pins = [
    Pin(8, Pin.IN, Pin.PULL_UP),
    Pin(9, Pin.IN, Pin.PULL_UP),
    Pin(20, Pin.IN, Pin.PULL_UP),
    Pin(21, Pin.IN, Pin.PULL_UP)
]

# 4x4 Keypad Layout Map
keys = [
    ['1', '2', '3', 'A'],
    ['4', '5', '6', 'B'],
    ['7', '8', '9', 'C'],
    ['*', '0', '#', 'D']
]

def scan_keypad():
    pressed_key = None
    
    # Multiplex scanning loop: drive one row LOW at a time
    for r_idx in range(4):
        row_pins[r_idx].value(0) # Drive active row LOW
        
        # Check all column pins
        for c_idx in range(4):
            if col_pins[c_idx].value() == 0: # Connection detected
                pressed_key = keys[r_idx][c_idx]
                # Wait until the key is released (debounce)
                while col_pins[c_idx].value() == 0:
                    utime.sleep_ms(10)
                    
        row_pins[r_idx].value(1) # Return row to HIGH
        
    return pressed_key

print("Keypad scanner active. Press any key!")

while True:
    key = scan_keypad()
    if key is not None:
        print("Key Pressed:", key)
    utime.sleep_ms(50) # Polling cycle speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **4x4 Keypad** onto the canvas.
2. Connect Row pins to **GP2, GP3, GP6, GP7**, and Column pins to **GP8, GP9, GP20, GP21**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the keypad keys on the canvas and observe the terminal output.

## Expected Output
```
Keypad scanner active. Press any key!
Key Pressed: 5
Key Pressed: A
Key Pressed: *
```

## Expected Canvas Behavior
- The serial terminal prints the character matching the key clicked on the keypad component.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `row_pins[r_idx].value(0)` | Pulls the active row LOW so that pressing a key will pull the connected column line LOW. |
| `col_pins[c_idx].value() == 0` | Detects a key press when a column input line is pulled LOW by the active row. |
| `while col_pins[c_idx].value() == 0` | Holds execution until the key is released to prevent duplicate readings. |

## Hardware & Safety Concept: Matrix Keypad Multiplexing
Connecting 16 individual switches directly to a microcontroller would require 16 GPIO pins. Keypad matrices use **multiplexing** (wiring keys in rows and columns) to read 16 switches using only **8 GPIO pins** (4 rows + 4 columns). This saves valuable GPIO pins on the microcontroller for other sensors and actuators.

## Try This! (Challenges)
1. **Buzzer Feedback**: Connect a buzzer on GP15 and sound a short beep when a key is pressed.
2. **Password Latch**: Store key presses in a list. When the correct passcode is entered (e.g. `1 2 3 4 #`), light up an LED on GP14.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Multiple keys are detected at once | Ghosting issue | Ensure keys are pressed one at a time. Do not hold multiple keys down together. |
| Keys in a specific row/col do not work | Broken connection | Check the wiring for that specific row/column pin. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [03 - Pico Button LED](03-pico-button-led.md)
- [13 - Pico Button Counter](13-pico-button-counter.md)
