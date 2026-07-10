# 124 - Pico Obstacle Avoidance Robot HC-SR04 L298N

Build a two-wheel obstacle avoidance robot that automatically drives forward, detects obstacles with an HC-SR04 sensor, and executes a reverse-and-turn escape maneuver when the path is blocked.

## Goal
Learn how to implement autonomous reactive navigation using an HC-SR04 ultrasonic sensor and an L298N dual motor driver, building a robot that drives forward and avoids obstacles without human input in MicroPython.

## What You Will Build
An autonomous obstacle-avoidance robot:
- **HC-SR04 Ultrasonic Sensor (GP16 TRIG, GP17 ECHO)**: Measures distance ahead.
- **L298N Motor Driver (GP10-GP13)**: Drives two DC motors for forward, reverse, and turning.
- **LED (GP15)**: Flashes when an obstacle is detected.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `hcsr04` | Yes | Yes |
| L298N Dual Motor Driver | `l298n` | Yes | Yes |
| DC Motor × 2 (robot wheels) | `dc_motor` | Yes | Yes |
| LED (obstacle indicator) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 | VCC | 5V (VBUS) | Red | Sensor power (requires 5V) |
| HC-SR04 | GND | GND | Black | Ground reference |
| HC-SR04 | TRIG | GP16 | Orange | Trigger pulse output |
| HC-SR04 | ECHO | GP17 | Yellow | Echo pulse input (use 1kΩ/2kΩ divider) |
| L298N Driver | ENA | GP10 | Orange | Left motor PWM speed |
| L298N Driver | IN1 | GP11 | Yellow | Left motor direction A |
| L298N Driver | IN2 | GP12 | Green | Left motor direction B |
| L298N Driver | ENB | GP13 | Orange | Right motor PWM speed |
| L298N Driver | IN3 | GP14 | Yellow | Right motor direction A |
| L298N Driver | IN4 | GP15 | Green | Right motor direction B |
| Obstacle LED | Anode (+, via 330 Ω) | GP2 | Orange | Obstacle detected indicator |
| Obstacle LED | Cathode (−) | GND | Black | Ground return |
| L298N Driver | VCC / GND | Battery+ / GND | — | Motor power supply (6–12V) |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The HC-SR04 ECHO pin outputs 5V logic. Add a 1kΩ/2kΩ voltage divider to bring it to 3.3V for the Pico GPIO. Connect the L298N to a separate 6V–9V motor supply. All grounds (Pico, L298N, HC-SR04) must be connected together.

## Code
```python
from machine import Pin, PWM
import utime
import machine

# HC-SR04
trig = Pin(16, Pin.OUT)
echo = Pin(17, Pin.IN)

# Left Motor: ENA=GP10, IN1=GP11, IN2=GP12
ena = PWM(Pin(10))
in1 = Pin(11, Pin.OUT)
in2 = Pin(12, Pin.OUT)
ena.freq(1000)

# Right Motor: ENB=GP13, IN3=GP14, IN4=GP15
enb = PWM(Pin(13))
in3 = Pin(14, Pin.OUT)
in4 = Pin(15, Pin.OUT)
enb.freq(1000)

obs_led = Pin(2, Pin.OUT)

# Motor speed (0 - 65535)
FULL_SPEED = 50000
TURN_SPEED = 45000
STOP_SPEED = 0

OBSTACLE_CM = 25  # Stop and avoid if closer than this

def measure_distance():
    trig.value(0)
    utime.sleep_us(2)
    trig.value(1)
    utime.sleep_us(10)
    trig.value(0)
    duration = machine.time_pulse_us(echo, 1, 30000)
    if duration < 0:
        return 999.0
    return duration / 58.0

def forward():
    in1.value(1); in2.value(0)  # Left motor forward
    in3.value(1); in4.value(0)  # Right motor forward
    ena.duty_u16(FULL_SPEED)
    enb.duty_u16(FULL_SPEED)

def backward():
    in1.value(0); in2.value(1)  # Left motor reverse
    in3.value(0); in4.value(1)  # Right motor reverse
    ena.duty_u16(FULL_SPEED)
    enb.duty_u16(FULL_SPEED)

def turn_right():
    in1.value(1); in2.value(0)  # Left motor forward
    in3.value(0); in4.value(1)  # Right motor reverse
    ena.duty_u16(TURN_SPEED)
    enb.duty_u16(TURN_SPEED)

def stop():
    ena.duty_u16(STOP_SPEED)
    enb.duty_u16(STOP_SPEED)

print("Obstacle avoidance robot active.")

while True:
    dist = measure_distance()
    
    if dist < OBSTACLE_CM:
        # Obstacle detected — execute escape sequence
        obs_led.value(1)
        print("Obstacle at {:.1f} cm — evading!".format(dist))
        
        stop()
        utime.sleep_ms(200)
        
        backward()
        utime.sleep_ms(600)
        
        stop()
        utime.sleep_ms(200)
        
        turn_right()
        utime.sleep_ms(500)
        
        stop()
        utime.sleep_ms(200)
        obs_led.value(0)
    else:
        # Path clear — drive forward
        forward()
        utime.sleep_ms(100)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-SR04**, **L298N Driver**, **two DC Motors**, and **LED** onto the canvas.
2. Connect HC-SR04 TRIG to **GP16**, ECHO to **GP17**. Connect L298N to **GP10–GP15**. Connect LED to **GP2**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Decrease the HC-SR04 slider below 25 cm to trigger the avoidance maneuver and observe the motor components change direction.

## Expected Output
```
Obstacle avoidance robot active.
Obstacle at 18.3 cm — evading!
```

## Expected Canvas Behavior
- Both DC motor components spin when driving forward. When the HC-SR04 slider is brought below 25 cm, both motors reverse, then one reverses while the other drives forward (right turn).

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `in1.value(1); in2.value(0)` | Sets the L298N direction pins for the left motor to forward. Reversing the pin states reverses the motor. |
| `backward() → turn_right()` | Sequential escape: reverse to clear the obstacle, then turn right to find a new path. |

## Hardware & Safety Concept: Reactive vs Deliberative Navigation
This robot uses **reactive navigation**: it immediately responds to sensor input with a preprogrammed avoidance behaviour. More sophisticated robots use **deliberative navigation** (path planning, maps) to choose the optimal evasion route. Reactive systems are simpler but can get trapped in corners. Commercial robot vacuums combine both methods.

## Try This! (Challenges)
1. **Left vs Right Decision**: Add a second HC-SR04 on GP18/GP19 pointed to the side, and choose to turn left or right based on which side has more space.
2. **Speed Ramping**: Ramp the motor speed from 0 to FULL_SPEED over 200 ms at startup to prevent wheel slip.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| One motor runs, one doesn't | L298N direction pin swap | Swap IN1/IN2 or IN3/IN4 for the non-spinning motor. |
| Robot always avoids | Threshold set too large | Lower `OBSTACLE_CM` to 15 cm and verify distance readings in the serial monitor. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [61 - Pico DC Motor L298N](61-pico-dc-motor-l298n.md)
- [108 - Pico Ultrasonic Distance LCD](108-pico-ultrasonic-distance-lcd.md)
- [115 - Pico Parking Sensor HC-SR04 Buzzer LCD](115-pico-parking-sensor-hcsr04-buzzer-lcd.md)
