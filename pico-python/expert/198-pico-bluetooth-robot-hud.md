# 198 - Pico Bluetooth Robot HUD Console

Build a remote-controlled mobile robot that parses movement directions over Bluetooth, drives DC motors, and uses an ultrasonic sensor to override controls if a collision is imminent.

## Goal
Learn how to parse serial commands over UART, implement safety overrides based on analog distance limits, drive DC motors with direction mapping, and update status readouts on an I2C LCD in MicroPython.

## What You Will Build
A Bluetooth-controlled robot with collision avoidance:
- **HC-05 Bluetooth Module (GP0, GP1)**: Receives movement characters (`F`, `B`, `L`, `R`, `S`) and transmits telemetry logs back.
- **L298N Motor Driver (GP10-GP13)**: Drives left and right DC motors.
- **HC-SR04 Ultrasonic Sensor (GP14, GP15)**: Monitors forward path clearance.
- **16x2 I2C LCD (GP4, GP5)**: Displays active speed, current distance, and control override states.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| L298N Motor Driver + Motors | `l298n` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `hcsr04` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-05 | TXD | GP1 (UART0 RX) | Yellow | BT transmit to Pico RX |
| HC-05 | RXD | GP0 (UART0 TX) | Orange | Pico TX to BT receive |
| HC-05 | VCC / GND | 3.3V / GND | Red / Black | Module power lines |
| Left Motor | ENA (Speed) | GP10 | Orange | Left motor PWM control |
| Left Motor | IN1 (Direction) | GP11 | Yellow | Left motor direction |
| Right Motor | ENB (Speed) | GP12 | Green | Right motor PWM control |
| Right Motor | IN3 (Direction) | GP13 | Blue | Right motor direction |
| HC-SR04 | TRIG / ECHO | GP14 / GP15 | Yellow / Blue | Obstacle detection sensor |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Ground reference |

> **Wiring tip:** Connect HC-05 TXD to GP1 and RXD to GP0. The L298N connects to GP10-GP13. The HC-SR04 connects to GP14/GP15. The LCD uses GP4/GP5.

## Code
```python
from machine import Pin, PWM, UART, I2C
import utime, machine
from machine_lcd import I2cLcd

# DC Motors setup
motor_l_spd = PWM(Pin(10)); motor_l_spd.freq(1000)
motor_l_dir = Pin(11, Pin.OUT)
motor_r_spd = PWM(Pin(12)); motor_r_spd.freq(1000)
motor_r_dir = Pin(13, Pin.OUT)

# HC-SR04 Rangefinder pins
trig = Pin(14, Pin.OUT)
echo = Pin(15, Pin.IN)

# Bluetooth UART0
bt = UART(0, baudrate=9600, tx=Pin(0), rx=Pin(1))

# I2C LCD
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Global variables
distance_cm = 400.0
cmd_state = "STOP"
speed = 0

def get_distance():
    trig.value(0)
    utime.sleep_us(2)
    trig.value(1)
    utime.sleep_us(10)
    trig.value(0)
    
    duration = machine.time_pulse_us(echo, 1, 30000)
    if duration < 0:
        return 400.0
    return duration / 58.0

def set_motors(left_speed, left_dir, right_speed, right_dir):
    """Sets motor speeds (0-100) and directions (1=forward, 0=reverse)."""
    motor_l_dir.value(left_dir)
    motor_r_dir.value(right_dir)
    motor_l_spd.duty_u16(int(left_speed * 655.35))
    motor_r_spd.duty_u16(int(right_speed * 655.35))

def stop_robot():
    set_motors(0, 1, 0, 1)

lcd.clear()
lcd.putstr("Bluetooth Robot\nConsole Ready...")
bt.write("Robot HUD Online. Press commands F, B, L, R, S.\r\n")
utime.sleep(1.5)

print("Bluetooth robot console active.")

while True:
    # 1. Measure forward path clearance
    distance_cm = get_distance()
    
    # 2. Check for incoming Bluetooth commands
    if bt.any():
        cmd = bt.read(1)
        
        # Process commands only if path is clear or if reversing/stopping
        if cmd == b'F':
            if distance_cm >= 20.0:
                set_motors(65, 1, 65, 1)  # Forward
                cmd_state = "FORWARD"
                speed = 65
                bt.write("ACK: FORWARD\r\n")
            else:
                bt.write("ERR: Path blocked!\r\n")
        elif cmd == b'B':
            set_motors(65, 0, 65, 0)      # Reverse
            cmd_state = "REVERSE"
            speed = 65
            bt.write("ACK: REVERSE\r\n")
        elif cmd == b'L':
            set_motors(60, 0, 60, 1)      # Spin Left
            cmd_state = "LEFT   "
            speed = 60
            bt.write("ACK: LEFT\r\n")
        elif cmd == b'R':
            set_motors(60, 1, 60, 0)      # Spin Right
            cmd_state = "RIGHT  "
            speed = 60
            bt.write("ACK: RIGHT\r\n")
        elif cmd == b'S':
            stop_robot()
            cmd_state = "STOPPED"
            speed = 0
            bt.write("ACK: STOPPED\r\n")

    # 3. Collision Avoidance Safety Override
    # If driving forward and an obstacle appears, force an immediate stop
    if cmd_state == "FORWARD" and distance_cm < 20.0:
        stop_robot()
        cmd_state = "SAFE:ST"
        speed = 0
        bt.write("WARNING: Collision Avoidance Override Activated!\r\n")
        print(">> COLLISION AVOIDANCE ACTIVATED!")

    # 4. Update LCD Screen
    lcd.clear()
    lcd.putstr("Stat:{} S:{}\n".format(cmd_state, speed))
    lcd.putstr("Dist: {:.1f} cm".format(distance_cm))
    
    utime.sleep_ms(100)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-05**, **two DC Motors**, **L298N**, **HC-SR04**, and **I2C LCD** onto the canvas.
2. Connect HC-05 to **GP0/GP1**, L298N to **GP10-GP13**, HC-SR04 to **GP14/GP15**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. In MbedO, type `'F'` into the HC-05 Bluetooth simulation input. Observe the motors spin forward.
5. Slide the HC-SR04 sensor distance below 20 cm. Verify that the motors stop immediately and the LCD updates to `Stat:SAFE:ST`.

## Expected Output
Terminal (Bluetooth):
```
Robot HUD Online. Press commands F, B, L, R, S.
ACK: FORWARD
WARNING: Collision Avoidance Override Activated!
```

## Expected Canvas Behavior
* Boot: LCD reads `Bluetooth Robot` / `Console Ready...` for 1.5 seconds.
* Send `'F'` on BT: Motors spin forward. LCD reads `Stat:FORWARD S:65` / `Dist: 400.0 cm`.
* Slide rangefinder below 20 cm: Motors stop spinning instantly. LCD reads `Stat:SAFE:ST` and the warning prints over Bluetooth.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `bt.read(1)` | Reads exactly 1 byte from the Bluetooth UART buffer to fetch the command character. |
| `cmd_state == "FORWARD" and distance_cm < 20.0` | Logical evaluation that overrides motor states if the robot is moving toward an obstacle. |

## Hardware & Safety Concept: Sensor-Assisted Safety Overrides
Wireless control channels (like Bluetooth or Wi-Fi) have latency and are prone to connection dropouts. If a robot is driving forward and the wireless link fails, the user cannot send a stop command, resulting in a collision. Integrating local, autonomous safety sensors (like the HC-SR04) on the microcontroller ensures the robot can override manual controls and stop itself if an obstacle is detected, protecting the hardware.

## Try This! (Challenges)
1. **Auditory Alarm**: Connect a buzzer on GP16 and beep it slowly when reversing (`B`), and rapidly during a safety override stop.
2. **Reverse Warning Ticker**: Write code that prints the current distance back to the Bluetooth client once per second only while the robot is moving.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot doesn't move when command is sent | Baud rate mismatch | Ensure your HC-05 module is configured to the standard 9600 baud rate in both code and hardware settings. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [114 - Pico BT Robot with Sensor](../advanced/114-pico-bt-robot-with-sensor.md)
- [127 - Pico Bluetooth Robot HC05 L298N LCD](../advanced/127-pico-bluetooth-robot-hc05-l298n-lcd.md)
