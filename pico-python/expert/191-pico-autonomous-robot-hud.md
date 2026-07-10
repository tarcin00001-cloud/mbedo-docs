# 191 - Pico Autonomous Robot HUD

Build an autonomous obstacle-avoidance mobile robot that runs its navigation and rangefinder checks on Core 0 while executing a real-time OLED HUD telemetry console on Core 1 in parallel.

## Goal
Learn how to implement dual-core task distribution on the RP2040, offload slow visual render routines from critical motor-control loops, and share data using thread-safe variables.

## What You Will Build
A dual-core mobile robot system:
- **Core 0 (Navigation Core)**: Polls the HC-SR04 ultrasonic sensor (GP14, GP15) and controls the L298N DC motor driver (GP10-GP13) to execute obstacle avoidance.
- **Core 1 (HUD Telemetry Core)**: Drives the SSD1306 OLED (GP4, GP5) to render a live status dashboard and obstacle alarm indicator.
- **L298N Motor Driver (GP10-GP13)**: Directs left and right DC drive motors.
- **HC-SR04 Sensor (GP14, GP15)**: Scans for obstacles ahead of the robot.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| SSD1306 OLED Display | `oled` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `hcsr04` | Yes | Yes |
| L298N Motor Driver + Motors | `l298n` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 | TRIG / ECHO | GP14 / GP15 | Yellow / Blue | Obstacle detection sensor |
| HC-SR04 | VCC / GND | 5V / GND | Red / Black | Sensor power lines |
| Left Motor | ENA (Speed) | GP10 | Orange | Left motor PWM control |
| Left Motor | IN1 (Direction) | GP11 | Yellow | Left motor direction |
| Right Motor | ENB (Speed) | GP12 | Green | Right motor PWM control |
| Right Motor | IN3 (Direction) | GP13 | Blue | Right motor direction |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The L298N motor driver requires external battery power (e.g. 6V to 9V) to drive the motors. Connect only the L298N GND back to the Pico GND to create a common reference.

## Code
```python
import _thread
from machine import Pin, PWM, I2C
import utime, machine, ssd1306

# DC Motors setup
motor_l_spd = PWM(Pin(10)); motor_l_spd.freq(1000)
motor_l_dir = Pin(11, Pin.OUT)
motor_r_spd = PWM(Pin(12)); motor_r_spd.freq(1000)
motor_r_dir = Pin(13, Pin.OUT)

# HC-SR04 Rangefinder pins
trig = Pin(14, Pin.OUT)
echo = Pin(15, Pin.IN)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

# Shared multi-core variables
distance_cm = 400.0
robot_state = "STOP"
lock = _thread.allocate_lock()

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

def core1_hud_thread():
    """Background thread running on Core 1: drives the OLED dashboard."""
    global distance_cm, robot_state
    print("[Core 1] Telemetry HUD thread active.")
    
    while True:
        # 1. Safely read shared variables
        lock.acquire()
        d = distance_cm
        state = robot_state
        lock.release()
        
        # 2. Draw HUD Display
        oled.fill(0)
        
        # Header banner
        oled.fill_rect(0, 0, 128, 14, 1)
        oled.text("AUTONOMOUS HUD", 10, 3, 0)
        
        oled.text("State: {}".format(state), 10, 20, 1)
        oled.text("Dist : {:.1f} cm".format(d), 10, 34, 1)
        
        # Warning alert bar
        if d < 20.0:
            oled.fill_rect(10, 48, 108, 12, 1)
            oled.text("!! DANGER !!", 20, 50, 0)
        else:
            oled.rect(10, 48, 108, 12, 1)
            oled.text("PATH CLEAR", 26, 50, 1)
            
        oled.rect(0, 0, 128, 64, 1)
        oled.show()
        
        # Render at 10 Hz
        utime.sleep_ms(100)

# Start Core 1 HUD thread
_thread.start_new_thread(core1_hud_thread, ())

print("[Core 0] Navigation loop active.")
utime.sleep(1.0)  # Wait for sensor stabilization

while True:
    # 1. Measure distance
    d = get_distance()
    
    # 2. Update navigation rules and motor states
    # If path is clear: Drive Forward
    if d >= 20.0:
        set_motors(60, 1, 60, 1) # Left Forward, Right Forward
        lock.acquire()
        distance_cm = d
        robot_state = "FORWARD "
        lock.release()
    # If obstacle is detected: Stop, Reverse, Turn
    else:
        # Stop
        set_motors(0, 1, 0, 1)
        lock.acquire()
        distance_cm = d
        robot_state = "STOPPED "
        lock.release()
        utime.sleep_ms(300)
        
        # Reverse
        set_motors(50, 0, 50, 0) # Left Reverse, Right Reverse
        lock.acquire()
        robot_state = "REVERSE "
        lock.release()
        utime.sleep_ms(600)
        
        # Turn Right (Left forward, Right reverse)
        set_motors(60, 1, 60, 0)
        lock.acquire()
        robot_state = "TURNING "
        lock.release()
        utime.sleep_ms(500)
        
        # Stop again
        set_motors(0, 1, 0, 1)
        utime.sleep_ms(200)

    utime.sleep_ms(60) # Settle delay before next reading
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **SSD1306 OLED**, **HC-SR04 Sensor**, **L298N**, and **two DC Motors** onto the canvas.
2. Connect HC-SR04 to **GP14/GP15**, L298N inputs to **GP10-GP13**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the ultrasonic sensor slider. Slide it below 20 cm. Observe the motor direction change on the canvas and verify that the OLED display reacts instantly.

## Expected Output
Terminal:
```
[Core 1] Telemetry HUD thread active.
[Core 0] Navigation loop active.
```

## Expected Canvas Behavior
* Normal state (dist > 20 cm): Motors spin forward. OLED reads `State: FORWARD`.
* Obstacle detected (dist < 20 cm): Motors stop, spin backwards, spin in opposite directions (turn), then resume forward.
* OLED changes from `PATH CLEAR` to `!! DANGER !!` instantly when obstacle distance is breached.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `_thread.start_new_thread(core1_hud_thread, ())` | Spawns the telemetry display updates onto Core 1, freeing Core 0 to perform real-time navigation. |
| `lock.acquire() / release()` | Synchronizes access to `distance_cm` and `robot_state` to prevent cross-core data corruption. |

## Hardware & Safety Concept: Decoupling Motor Controls from Visuals
Driving motors and processing ultrasonic echo pulses require precise timing. Drawing to I2C screens is a relatively slow operation (taking up to 30ms to transmit 1024 bytes at 400kHz). If they run in the same thread, the robot will freeze for 30ms during every screen draw, making it slow to detect obstacles and prone to collisions. Running them on separate cores resolves this lag entirely.

## Try This! (Challenges)
1. **Critical Crash Beeper**: Connect a buzzer on GP16 and sound a fast warning yelp when the robot is in `REVERSE` or `TURNING` states.
2. **Speed Dial**: Connect a potentiometer on GP26 and map its value to adjust the default robot cruising speed between 40% and 80%.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The robot stops responding or the OLED freezes | Thread collision | Ensure that only Core 1 accesses the OLED object `oled`. Accessing I2C displays from both cores simultaneously will lock the I2C bus. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [124 - Pico Obstacle Avoidance Robot HC-SR04 L298N](../advanced/124-pico-obstacle-avoidance-robot-hcsr04-l298n.md)
- [181 - Pico Multithreaded Core Execution](181-pico-multithreaded-core-execution.md)
