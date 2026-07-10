# 134 - Pico Line Follower Robot

Build a two-wheel line-following robot that uses two digital IR line sensors to detect and track a dark line on a light surface, adjusting left and right motor speeds to stay on the line.

## Goal
Learn how to read two digital IR reflectance sensors, implement differential steering logic to correct left or right deviations, and drive a two-wheel robot using an L298N dual motor driver in MicroPython.

## What You Will Build
A line-following robot:
- **IR Line Sensor Left (GP14)**: Detects if the left wheel has strayed off the line.
- **IR Line Sensor Right (GP15)**: Detects if the right wheel has strayed off the line.
- **L298N Motor Driver (GP10-GP13)**: Drives left and right DC motors with differential speed corrections.
- **LED (GP2)**: Flashes when the robot loses the line on both sensors.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| IR Line Sensor Module × 2 | `button` | Yes (digital output) | Yes |
| L298N Dual Motor Driver | `l298n` | Yes | Yes |
| DC Motor × 2 | `dc_motor` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| IR Sensor Left | VCC | 3.3V (3V3) | Red | Sensor power |
| IR Sensor Left | GND | GND | Black | Ground reference |
| IR Sensor Left | OUT | GP14 | Blue | LOW = line detected (dark surface) |
| IR Sensor Right | VCC | 3.3V (3V3) | Red | Sensor power |
| IR Sensor Right | GND | GND | Black | Ground reference |
| IR Sensor Right | OUT | GP15 | Yellow | LOW = line detected (dark surface) |
| L298N | ENA | GP10 | Orange | Left motor PWM speed |
| L298N | IN1 | GP11 | Yellow | Left motor direction A |
| L298N | IN2 | GP12 | Green | Left motor direction B |
| L298N | ENB | GP13 | Orange | Right motor PWM speed |
| L298N | IN3 | GP4 | Yellow | Right motor direction A |
| L298N | IN4 | GP5 | Green | Right motor direction B |
| LED | Anode (+, via 330 Ω) | GP2 | Orange | Lost-line indicator |
| LED | Cathode (−) | GND | Black | Ground return |
| L298N | VCC / GND | Battery / GND | — | Motor power supply (6–9V) |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** IR line sensors output LOW when detecting a dark line and HIGH over a light surface. Connect Left sensor OUT to GP14, Right sensor OUT to GP15. Connect L298N direction pins to GP11/GP12 (left) and GP4/GP5 (right). All grounds are shared.

## Code
```python
from machine import Pin, PWM
import utime

# IR line sensors (LOW = line detected)
sensor_l = Pin(14, Pin.IN, Pin.PULL_UP)
sensor_r = Pin(15, Pin.IN, Pin.PULL_UP)

# Left motor
ena = PWM(Pin(10)); ena.freq(1000)
in1 = Pin(11, Pin.OUT)
in2 = Pin(12, Pin.OUT)

# Right motor
enb = PWM(Pin(13)); enb.freq(1000)
in3 = Pin(4, Pin.OUT)
in4 = Pin(5, Pin.OUT)

lost_led = Pin(2, Pin.OUT)

FULL   = 52000
SLOW   = 30000
STOP   = 0

def motor_left(speed, fwd=True):
    in1.value(1 if fwd else 0)
    in2.value(0 if fwd else 1)
    ena.duty_u16(speed)

def motor_right(speed, fwd=True):
    in3.value(1 if fwd else 0)
    in4.value(0 if fwd else 1)
    enb.duty_u16(speed)

def stop_all():
    ena.duty_u16(STOP)
    enb.duty_u16(STOP)

stop_all()
print("Line follower active. Place on track.")

while True:
    # LOW = on line; HIGH = off line
    left_on  = (sensor_l.value() == 0)
    right_on = (sensor_r.value() == 0)

    if left_on and right_on:
        # Both on line — drive straight
        motor_left(FULL)
        motor_right(FULL)
        lost_led.value(0)
        print("STRAIGHT")

    elif left_on and not right_on:
        # Right sensor off line — veer right
        # Slow left motor, speed right motor to curve right
        motor_left(SLOW)
        motor_right(FULL)
        lost_led.value(0)
        print("VEER RIGHT")

    elif not left_on and right_on:
        # Left sensor off line — veer left
        motor_left(FULL)
        motor_right(SLOW)
        lost_led.value(0)
        print("VEER LEFT")

    else:
        # Both off line — stop and signal lost
        stop_all()
        lost_led.value(1)
        print("LINE LOST — stopping")

    utime.sleep_ms(50)
```

## What to Click in MbedO
1. Drag **Pico**, **two IR Sensor Buttons**, **L298N Driver**, **two DC Motors**, and **LED** onto the canvas.
2. Connect Left IR to **GP14**, Right IR to **GP15**, L298N to **GP10–GP13** and **GP4/GP5**, and LED to **GP2**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click Left Button to simulate left-on-line, Right Button for right-on-line. Both OFF = lost. Both ON = straight.

## Expected Output
```
Line follower active. Place on track.
STRAIGHT
VEER RIGHT
LINE LOST — stopping
```

## Expected Canvas Behavior
- Both IR buttons pressed: both DC motors run at full speed. Only left pressed: right motor speeds up and left slows (veers right). Only right pressed: left motor speeds up. Neither pressed: motors stop and LED lights.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `sensor_l.value() == 0` | Active-low detection: sensor outputs LOW when it sees the dark line surface. |
| `motor_left(SLOW)` / `motor_right(FULL)` | Differential speed correction: slowing one motor curves the robot toward the line. |

## Hardware & Safety Concept: Differential Drive Steering
Two-wheel robots steer by varying the speed or direction of each wheel independently — called **differential drive**. Turning is produced by running one wheel faster than the other, without a steering mechanism. Sharp turns are achieved by running motors in opposite directions. This is the same principle used in tank tracks and wheelchair drive systems.

## Try This! (Challenges)
1. **PID Controller**: Replace the bang-bang correction with a proportional controller: error = left_reading − right_reading, correction = Kp × error applied to motor speeds.
2. **Speed Increase on Straight**: When both sensors detect the line, increase speed to FULL+10% for faster straight-line performance.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot always veers one direction | One sensor inverted | Swap the sensor wires for the drifting side, or adjust the sensor module's onboard sensitivity trim pot. |
| Robot oscillates badly | SLOW speed too low | Increase `SLOW` from 30000 to 40000 to reduce correction overshoot. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [124 - Pico Obstacle Avoidance Robot HC-SR04 L298N](124-pico-obstacle-avoidance-robot-hcsr04-l298n.md)
- [127 - Pico Bluetooth Robot HC-05 L298N LCD](127-pico-bluetooth-robot-hc05-l298n-lcd.md)
