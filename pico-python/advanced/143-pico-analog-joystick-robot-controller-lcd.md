# 143 - Pico Analog Joystick Robot Controller LCD

Build a two-wheel robot controller that maps X/Y analog joystick inputs to differential drive motor speeds, allowing intuitive directional control with a single joystick, and displays the joystick position and motor output on an I2C LCD.

## Goal
Learn how to read a dual-axis analog joystick on two ADC channels, implement a dead-zone filter, convert polar joystick coordinates to differential motor speeds using arcade-drive mixing, and display the control state on an I2C character LCD in MicroPython.

## What You Will Build
A joystick-controlled differential drive robot:
- **Analog Joystick (GP26 X-axis, GP27 Y-axis, GP15 button)**: Controls robot direction and speed.
- **L298N Motor Driver (GP10-GP13)**: Drives left and right DC motors.
- **I2C 16x2 LCD (GP4, GP5)**: Displays X/Y values, direction label, and motor speed outputs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Analog Joystick Module | `potentiometer` | Yes (two separate sliders for X/Y) | Yes |
| L298N Dual Motor Driver | `l298n` | Yes | Yes |
| DC Motor × 2 | `dc_motor` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Joystick | VCC | 3.3V (3V3) | Red | Module power |
| Joystick | GND | GND | Black | Ground reference |
| Joystick | VRx (X-axis) | GP26 | Orange | Left-right axis ADC input |
| Joystick | VRy (Y-axis) | GP27 | Yellow | Forward-back axis ADC input |
| Joystick | SW (Button) | GP15 | Blue | Joystick click (optional — STOP) |
| L298N | ENA | GP10 | Orange | Left motor PWM speed |
| L298N | IN1 | GP11 | Yellow | Left motor direction A |
| L298N | IN2 | GP12 | Green | Left motor direction B |
| L298N | ENB | GP13 | Orange | Right motor PWM speed |
| L298N | IN3 | GP4 | Yellow | Right motor direction A |
| L298N | IN4 | GP5 | Green | Right motor direction B |
| I2C LCD | SDA / SCL | GP2 / GP3 | Shared Orange/Blue | I2C Bus 1 (avoids GP4/GP5 conflict with L298N) |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The L298N direction pins (IN3/IN4) use GP4/GP5, so the LCD moves to I2C Bus 1 (GP2/GP3). Joystick X → GP26, Y → GP27. Joystick button → GP15 as optional emergency stop. All grounds are shared.

## Code
```python
from machine import Pin, ADC, PWM, I2C
import utime
from machine_lcd import I2cLcd

# Joystick ADC inputs
joy_x = ADC(26)   # Left/Right
joy_y = ADC(27)   # Forward/Back
joy_btn = Pin(15, Pin.IN, Pin.PULL_UP)

# Left motor: ENA=GP10, IN1=GP11, IN2=GP12
ena = PWM(Pin(10)); ena.freq(1000)
in1 = Pin(11, Pin.OUT); in2 = Pin(12, Pin.OUT)

# Right motor: ENB=GP13, IN3=GP4, IN4=GP5
enb = PWM(Pin(13)); enb.freq(1000)
in3 = Pin(4, Pin.OUT); in4 = Pin(5, Pin.OUT)

# I2C Bus 1 for LCD (GP2/GP3)
i2c = I2C(1, sda=Pin(2), scl=Pin(3), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

DEAD_ZONE = 3000   # ADC units around centre (32767) ignored as noise
MAX_SPEED = 55000

def read_axis(adc):
    raw = adc.read_u16()
    val = raw - 32767          # Centre at zero (-32767 to +32768)
    if abs(val) < DEAD_ZONE:
        val = 0
    return val

def arcade_mix(fwd, turn):
    """Convert forward/turn to left/right motor values (arcade drive)."""
    left  = fwd + turn
    right = fwd - turn
    scale = max(abs(left), abs(right), 32767) / 32767
    if scale > 1.0:
        left  = int(left  / scale)
        right = int(right / scale)
    return left, right

def drive(left_val, right_val):
    # Left motor
    in1.value(1 if left_val >= 0 else 0)
    in2.value(0 if left_val >= 0 else 1)
    ena.duty_u16(min(MAX_SPEED, int(abs(left_val) * MAX_SPEED / 32767)))
    # Right motor
    in3.value(1 if right_val >= 0 else 0)
    in4.value(0 if right_val >= 0 else 1)
    enb.duty_u16(min(MAX_SPEED, int(abs(right_val) * MAX_SPEED / 32767)))

def stop_all():
    ena.duty_u16(0); enb.duty_u16(0)

lcd.clear(); lcd.putstr("Joystick Robot"); utime.sleep(1)
print("Joystick robot controller active.")

while True:
    if joy_btn.value() == 0:
        stop_all()
        lcd.clear(); lcd.putstr("STOPPED"); lcd.move_to(0, 1); lcd.putstr("BTN pressed")
        utime.sleep_ms(200)
        continue

    fwd  = read_axis(joy_y)   # Y = forward/back
    turn = read_axis(joy_x)   # X = left/right

    left_val, right_val = arcade_mix(fwd, turn)
    drive(left_val, right_val)

    # Direction label
    if abs(fwd) < DEAD_ZONE and abs(turn) < DEAD_ZONE:
        direction = "STOP   "
    elif fwd > 0 and abs(turn) < DEAD_ZONE:
        direction = "FORWARD"
    elif fwd < 0 and abs(turn) < DEAD_ZONE:
        direction = "REVERSE"
    elif fwd > 0 and turn > 0:
        direction = "FWD-RGT"
    elif fwd > 0 and turn < 0:
        direction = "FWD-LFT"
    else:
        direction = "TURNING"

    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("X:{:5d} Y:{:5d}".format(read_axis(joy_x), read_axis(joy_y)))
    lcd.move_to(0, 1)
    lcd.putstr(direction + " L:{:3d}%".format(abs(left_val) * 100 // 32767))

    utime.sleep_ms(50)
```

## What to Click in MbedO
1. Drag **Pico**, **two Potentiometers** (X and Y), **L298N**, **two DC Motors**, and **I2C LCD** onto the canvas.
2. Connect X-Pot to **GP26**, Y-Pot to **GP27**, L298N to **GP10-GP13** and **GP4/GP5**, LCD to **GP2/GP3** (I2C Bus 1).
3. Paste code, click **Run**. Slide the Y potentiometer forward to drive, X to steer.

## Expected Output
```
Joystick robot controller active.
X:    0 Y:15000   FORWARD L: 45%
X:-8000 Y:12000   FWD-LFT L: 62%
```

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `arcade_mix(fwd, turn)` | Arcade-drive mixing: combines a single forward/back and turn input into independent left/right motor values. |
| Dead zone filter | Values within `±DEAD_ZONE` of centre are treated as zero, preventing motor jitter when the joystick is released. |

## Hardware & Safety Concept: Arcade Drive Mixing
Arcade drive is the standard control scheme for two-wheel robots: one stick axis controls forward/backward speed and the other controls turning. The mixing algorithm adds the turn value to the left motor and subtracts it from the right motor, producing the correct differential for the desired direction.

## Try This! (Challenges)
1. **Throttle Curve**: Apply a cubic curve to the raw joystick value (`val³ / 32767²`) for finer control at low speeds.
2. **Speed Limit Button**: Add a third button on GP14 that halves `MAX_SPEED` for precision manoeuvring.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot drifts when joystick centred | Dead zone too small | Increase `DEAD_ZONE` from 3000 to 6000 to compensate for joystick offset. |
| One motor runs backwards | Motor wires reversed | Swap IN1/IN2 or IN3/IN4 for the affected motor channel. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [124 - Pico Obstacle Avoidance Robot HC-SR04 L298N](124-pico-obstacle-avoidance-robot-hcsr04-l298n.md)
- [127 - Pico Bluetooth Robot HC-05 L298N LCD](127-pico-bluetooth-robot-hc05-l298n-lcd.md)
