# 138 - Pico Stepper Motor CNC Positioner LCD

Build a stepper motor position controller that reads target position from a rotary encoder or potentiometer, drives a 28BYJ-48 stepper motor to the commanded position with a ULN2003 driver, and displays current and target position on an I2C LCD.

## Goal
Learn how to drive a 28BYJ-48 stepper motor using half-step sequencing on a ULN2003 driver, implement position tracking with step counting, and display current and target positions on an I2C character LCD in MicroPython.

## What You Will Build
A stepper motor position controller:
- **Potentiometer (GP26)**: Sets target position (mapped to 0–200 steps).
- **28BYJ-48 Stepper + ULN2003 (GP10-GP13)**: Drives to the commanded position.
- **I2C 16x2 LCD (GP4, GP5)**: Displays current step, target step, and direction of travel.
- **LED (GP15)**: Flashes while the motor is in motion.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| 28BYJ-48 Stepper Motor | `stepper` | Yes | Yes |
| ULN2003 Driver Board | `uln2003` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| ULN2003 | IN1 | GP10 | Orange | Stepper coil A |
| ULN2003 | IN2 | GP11 | Yellow | Stepper coil B |
| ULN2003 | IN3 | GP12 | Green | Stepper coil C |
| ULN2003 | IN4 | GP13 | Blue | Stepper coil D |
| ULN2003 | VCC | 5V (VBUS) | Red | Driver supply voltage |
| ULN2003 | GND | GND | Black | Ground reference |
| 28BYJ-48 | Coil connector | ULN2003 outputs | — | 5-pin JST connector |
| Potentiometer | Wiper (centre) | GP26 | Yellow | Target position input |
| Potentiometer | Left pin | GND | Black | Ground reference |
| Potentiometer | Right pin | 3.3V (3V3) | Red | High reference voltage |
| LED | Anode (+, via 330 Ω) | GP15 | Orange | Motion indicator |
| LED | Cathode (−) | GND | Black | Ground return |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the ULN2003 driver IN1-IN4 to GP10-GP13. The 28BYJ-48 plugs directly into the ULN2003 via its JST connector. Connect the potentiometer to GP26 and the LCD to GP4/GP5. The ULN2003 uses 5V from VBUS. All grounds are shared.

## Code
```python
from machine import Pin, ADC, I2C
import utime
from machine_lcd import I2cLcd

# ULN2003 stepper pins (IN1-IN4)
coils = [Pin(p, Pin.OUT) for p in (10, 11, 12, 13)]

# Half-step sequence (8 steps per electrical cycle)
HALF_STEP = [
    [1, 0, 0, 0],
    [1, 1, 0, 0],
    [0, 1, 0, 0],
    [0, 1, 1, 0],
    [0, 0, 1, 0],
    [0, 0, 1, 1],
    [0, 0, 0, 1],
    [1, 0, 0, 1],
]

pot       = ADC(26)
motion_led = Pin(15, Pin.OUT)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

MAX_STEPS   = 200
STEP_DELAY  = 2  # ms per half-step

current_pos = 0  # Tracked position in steps
seq_idx     = 0  # Current position in the half-step sequence

def step_motor(direction):
    """Advance motor one half-step. direction: +1 = CW, -1 = CCW."""
    global seq_idx
    seq_idx = (seq_idx + direction) % 8
    for i, c in enumerate(coils):
        c.value(HALF_STEP[seq_idx][i])

def release_coils():
    for c in coils:
        c.value(0)

def move_to(target):
    global current_pos
    direction = 1 if target > current_pos else -1
    while current_pos != target:
        step_motor(direction)
        current_pos += direction
        utime.sleep_ms(STEP_DELAY)
    release_coils()

lcd.clear(); lcd.putstr("Stepper Postnr")
utime.sleep(1)
print("Stepper positioner active. Turn pot to set target.")

while True:
    raw    = pot.read_u16()
    target = int(raw * MAX_STEPS / 65535)

    direction = "CW " if target > current_pos else "CCW" if target < current_pos else "---"

    # Update LCD before moving
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("Cur:{:4d} Tgt:{:4d}".format(current_pos, target))
    lcd.move_to(0, 1)
    lcd.putstr("Dir:{} Steps:{:4d}".format(direction, abs(target - current_pos)))

    if current_pos != target:
        motion_led.value(1)
        # Move one step at a time to allow LCD update
        d = 1 if target > current_pos else -1
        step_motor(d)
        current_pos += d
        if current_pos == target:
            release_coils()
            motion_led.value(0)
        utime.sleep_ms(STEP_DELAY)
    else:
        release_coils()
        motion_led.value(0)
        utime.sleep_ms(100)

    print("Pos:{:4d} Target:{:4d} Dir:{}".format(current_pos, target, direction))
```

## What to Click in MbedO
1. Drag **Pico**, **Potentiometer**, **Stepper Motor** (28BYJ-48 + ULN2003), **LED**, and **I2C LCD** onto the canvas.
2. Connect Potentiometer to **GP26**, ULN2003 IN1-IN4 to **GP10-GP13**, LED to **GP15**, and LCD to **GP4/GP5**. Connect power and GND.
3. Paste code, select **MicroPython** mode, click **Run**.
4. Slide the potentiometer and observe the stepper motor moving to the target position. LCD shows current vs target and direction.

## Expected Output
```
Stepper positioner active. Turn pot to set target.
Pos:   0 Target: 150 Dir:CW
Pos: 150 Target: 150 Dir:---
Pos: 150 Target:  60 Dir:CCW
```

## Expected Canvas Behavior
- As the potentiometer slider changes, the stepper motor component steps in the corresponding direction. The LED flashes while moving. The LCD shows the real-time positions and step count remaining.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `HALF_STEP` sequence | Defines the 8-phase half-step excitation pattern. Half-stepping doubles angular resolution vs full-step and produces smoother motion. |
| `seq_idx = (seq_idx + direction) % 8` | Advances the half-step sequence index forward (+1) or backward (-1), wrapping around at 8. |

## Hardware & Safety Concept: Coil Release After Move
Stepper motors hold position by keeping coils energized. Leaving coils powered when stationary wastes energy and heats the motor. Calling `release_coils()` de-energizes all coils after reaching the target position. This is safe for low-torque positioning tasks but should not be used where the load can back-drive the motor.

## Try This! (Challenges)
1. **Speed Ramp**: Gradually decrease `STEP_DELAY` from 10 ms to 2 ms over the first 20 steps to prevent missed steps at startup.
2. **Limit Switches**: Add two buttons on GP0 and GP1 as home and end-stop limit switches that reset `current_pos = 0` on contact.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor vibrates but does not rotate | Wrong half-step pin order | Swap IN1/IN2 or IN3/IN4 to match your specific ULN2003 board pinout. |
| Motor overheats | Coils not released | Ensure `release_coils()` is called after every move. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [60 - Pico Stepper Motor ULN2003](../intermediate/60-pico-stepper-motor-uln2003.md)
- [98 - Pico Stepper Motor Potentiometer LCD](../intermediate/98-pico-stepper-motor-potentiometer-lcd.md)
