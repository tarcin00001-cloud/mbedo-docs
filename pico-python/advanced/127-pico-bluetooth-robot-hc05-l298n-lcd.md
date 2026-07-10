# 127 - Pico Bluetooth Robot HC-05 L298N LCD

Build a Bluetooth-controlled two-wheel robot that receives directional commands from a smartphone app via an HC-05 module, drives the L298N motor driver accordingly, and displays the active command and distance reading on an I2C LCD.

## Goal
Learn how to receive UART characters from an HC-05 Bluetooth module, map command characters to motor direction functions, implement an auto-stop safety feature using an HC-SR04 sensor, and display status on an I2C LCD in MicroPython.

## What You Will Build
A Bluetooth-controlled robot:
- **HC-05 Bluetooth Module (GP0 TX, GP1 RX)**: Receives directional commands from a phone app.
- **L298N Motor Driver (GP10-GP13)**: Drives left and right DC motors.
- **HC-SR04 Ultrasonic Sensor (GP16, GP17)**: Auto-stops the robot if an obstacle is detected.
- **I2C 16x2 LCD (GP4, GP5)**: Displays the last received command and forward distance.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-05 Bluetooth Module | `hc05` | Yes | Yes |
| L298N Dual Motor Driver | `l298n` | Yes | Yes |
| DC Motor × 2 | `dc_motor` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `hcsr04` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-05 | TXD | GP1 (UART0 RX) | Yellow | BT data to Pico |
| HC-05 | RXD | GP0 (UART0 TX) | Orange | Pico data to BT (use 1kΩ divider to protect HC-05 from 3.3V) |
| HC-05 | VCC | 5V (VBUS) | Red | BT module power |
| HC-05 | GND | GND | Black | Ground reference |
| L298N | ENA | GP10 | Orange | Left motor PWM speed |
| L298N | IN1 | GP11 | Yellow | Left motor direction A |
| L298N | IN2 | GP12 | Green | Left motor direction B |
| L298N | ENB | GP13 | Orange | Right motor PWM speed |
| L298N | IN3 | GP14 | Yellow | Right motor direction A |
| L298N | IN4 | GP15 | Green | Right motor direction B |
| HC-SR04 | TRIG | GP16 | Orange | Ultrasonic trigger |
| HC-SR04 | ECHO | GP17 | Yellow | Ultrasonic echo (1kΩ/2kΩ divider required) |
| HC-SR04 | VCC | 5V (VBUS) | Red | Sensor power |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the HC-05 TXD to GP1 (UART0 RX) and RXD to GP0 (UART0 TX). Use a 1kΩ voltage divider on the HC-05 RXD line to reduce the 3.3V Pico TX to 3.3V safe for HC-05. The HC-SR04 ECHO requires a 1kΩ/2kΩ divider from 5V to 3.3V. All grounds are shared.

## Code
```python
from machine import Pin, PWM, UART, I2C
import utime
import machine
from machine_lcd import I2cLcd

# HC-05 Bluetooth on UART0 (GP0=TX, GP1=RX)
bt = UART(0, baudrate=9600, tx=Pin(0), rx=Pin(1))

# Left Motor: ENA=GP10, IN1=GP11, IN2=GP12
ena = PWM(Pin(10)); ena.freq(1000)
in1 = Pin(11, Pin.OUT)
in2 = Pin(12, Pin.OUT)

# Right Motor: ENB=GP13, IN3=GP14, IN4=GP15
enb = PWM(Pin(13)); enb.freq(1000)
in3 = Pin(14, Pin.OUT)
in4 = Pin(15, Pin.OUT)

# HC-SR04
trig = Pin(16, Pin.OUT)
echo = Pin(17, Pin.IN)

# I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

SPEED     = 50000
TURN_SPD  = 40000
SAFE_DIST = 20  # cm — auto-stop threshold

def measure_dist():
    trig.value(0); utime.sleep_us(2)
    trig.value(1); utime.sleep_us(10)
    trig.value(0)
    d = machine.time_pulse_us(echo, 1, 30000)
    return 999.0 if d < 0 else d / 58.0

def stop():
    ena.duty_u16(0); enb.duty_u16(0)

def forward():
    in1.value(1); in2.value(0)
    in3.value(1); in4.value(0)
    ena.duty_u16(SPEED); enb.duty_u16(SPEED)

def backward():
    in1.value(0); in2.value(1)
    in3.value(0); in4.value(1)
    ena.duty_u16(SPEED); enb.duty_u16(SPEED)

def turn_left():
    in1.value(0); in2.value(1)
    in3.value(1); in4.value(0)
    ena.duty_u16(TURN_SPD); enb.duty_u16(TURN_SPD)

def turn_right():
    in1.value(1); in2.value(0)
    in3.value(0); in4.value(1)
    ena.duty_u16(TURN_SPD); enb.duty_u16(TURN_SPD)

stop()
lcd.clear()
lcd.putstr("BT Robot Ready")
utime.sleep(1)

last_cmd = "S"
print("Bluetooth robot active. Send F/B/L/R/S commands.")

while True:
    dist = measure_dist()

    # Read BT command (non-blocking)
    if bt.any():
        cmd = bt.read(1).decode('utf-8', 'ignore').strip().upper()
        if cmd in ('F', 'B', 'L', 'R', 'S'):
            last_cmd = cmd
            print("CMD:", cmd, "| Dist:", dist)

    # Execute command (with obstacle override)
    if last_cmd == 'F' and dist > SAFE_DIST:
        forward()
    elif last_cmd == 'F' and dist <= SAFE_DIST:
        stop()  # Auto-stop
    elif last_cmd == 'B':
        backward()
    elif last_cmd == 'L':
        turn_left()
    elif last_cmd == 'R':
        turn_right()
    else:
        stop()

    # LCD
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("CMD: " + last_cmd + (" BLOCKED" if last_cmd == 'F' and dist <= SAFE_DIST else "       "))
    lcd.move_to(0, 1)
    lcd.putstr("Dist: {:.0f} cm     ".format(dist))

    utime.sleep_ms(80)
```

## What to Click in MbedO
1. Drag **Pico**, **HC-05**, **L298N**, **two DC Motors**, **HC-SR04**, and **I2C LCD** onto the canvas.
2. Wire per the table. Send commands F, B, L, R, S from the BT serial terminal.
3. Click **Run** and use a BT serial app (e.g. Serial Bluetooth Terminal) to control the robot.
4. Decrease the HC-SR04 slider below 20 cm while sending `F` — the robot auto-stops.

## Expected Output
```
Bluetooth robot active. Send F/B/L/R/S commands.
CMD: F | Dist: 45.0
CMD: F | Dist: 18.0   ← auto-stop kicks in
CMD: R | Dist: 45.0
```

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `bt.any()` | Returns the number of bytes available in the UART receive buffer without blocking. |
| `last_cmd == 'F' and dist <= SAFE_DIST` | Overrides the forward command with a stop when the ultrasonic sensor detects an obstacle. |

## Hardware & Safety Concept: Command Latching with Safety Override
The last received command is **latched** (stored in `last_cmd`) so the robot keeps moving without requiring the phone to continuously send the same character. The obstacle override is evaluated on every loop iteration, providing real-time safety regardless of the latched command.

## Try This! (Challenges)
1. **Speed Control**: Send digit characters `1`–`9` from the BT app to set motor speed from 10% to 90%.
2. **Buzzer Horn**: Connect a buzzer on GP2 and activate it when the `H` command is received.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot does not respond to BT | UART RX/TX swap | Ensure HC-05 TXD → GP1 (Pico RX) and HC-05 RXD → GP0 (Pico TX). |
| Auto-stop fires continuously | HC-SR04 timeout | Add voltage divider on ECHO and verify physical connection. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [124 - Pico Obstacle Avoidance Robot HC-SR04 L298N](124-pico-obstacle-avoidance-robot-hcsr04-l298n.md)
