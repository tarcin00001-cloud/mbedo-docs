# 197 - Pico Robotic Arm PID Position Controller

Build a closed-loop robotic joint position controller that reads analog feedback potentiometers, executes a Proportional-Integral-Derivative (PID) algorithm, and drives DC motors via an L298N to track targets.

## Goal
Learn how to implement a software PID position control loop, read analog joint position feedback, calculate error derivatives and integrals with anti-windup, and drive DC motors to track target paths in MicroPython.

## What You Will Build
A closed-loop robotic joint controller:
- **Joint 1 DC Motor (L298N: ENA GP10, IN1 GP11, IN2 GP12)**: Actuates joint 1.
- **Joint 1 Feedback Pot (GP26)**: Measures the actual physical angle of joint 1.
- **Target Setpoint Pot (GP27)**: Sets the desired target angle for joint 1.
- **SSD1306 OLED (GP4, GP5)**: Displays the target setpoint, actual position, error, and output duty cycle.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DC Motor | `dc_motor` | Yes | Yes (joint motor) |
| L298N Motor Driver | `l298n` | Yes | Yes |
| Potentiometers × 2 | `potentiometer` | Yes (two pots) | Yes (one target, one feedback) |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Feedback Pot | Wiper | GP26 | Yellow | Joint 1 actual position |
| Target Pot | Wiper | GP27 | Green | Desired target setpoint |
| Both Pots | Left / Right | 3.3V / GND | Red / Black | Voltage reference rails |
| L298N | ENA (PWM) | GP10 | Orange | Motor speed control |
| L298N | IN1 / IN2 | GP11 / GP12 | Yellow / Blue | Motor direction controls |
| L298N | Motor power | 6-12V external | Red | L298N motor supply |
| L298N | GND | GND | Black | Shared ground |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The Feedback Potentiometer must be physically coupled to the output shaft of the DC motor/gearbox to provide joint angle feedback.

## Code
```python
from machine import Pin, PWM, ADC, I2C
import utime, ssd1306

# L298N Pin setup
ena = PWM(Pin(10)); ena.freq(1000)
in1 = Pin(11, Pin.OUT)
in2 = Pin(12, Pin.OUT)

# Analog inputs
feedback_pot = ADC(26)  # actual position
target_pot = ADC(27)    # desired position

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

# PID Constants (Tuned for typical small gearbox motor)
Kp = 0.8
Ki = 0.05
Kd = 0.15

# Loop timing parameters
dt = 0.02  # 20 ms loop interval
last_error = 0.0
integral = 0.0

def set_motor(speed_pct):
    """Sets speed (-100 to 100). Negative values reverse direction."""
    # Clamp speed
    speed_pct = max(-100, min(100, speed_pct))
    
    # Deadband filter: ignore values below threshold to prevent humming
    if abs(speed_pct) < 10:
        in1.value(0); in2.value(0)
        ena.duty_u16(0)
        return
        
    if speed_pct >= 0:
        in1.value(1); in2.value(0)
        ena.duty_u16(int(speed_pct * 655.35))
    else:
        in1.value(0); in2.value(1)
        ena.duty_u16(int(abs(speed_pct) * 655.35))

oled.fill(0)
oled.text("PID Controller", 10, 20)
oled.text("Ready", 10, 36)
oled.show()
utime.sleep(1.0)

print("Joint PID position controller active.")

while True:
    # 1. Read targets and actual positions (mapped to 0-100 scale)
    raw_target = target_pot.read_u16()
    raw_actual = feedback_pot.read_u16()
    
    target_pos = raw_target * 100 // 65536
    actual_pos = raw_actual * 100 // 65536
    
    # 2. Execute PID calculations
    error = target_pos - actual_pos
    
    # Integral term with anti-windup clamping
    integral = max(-100, min(100, integral + error * dt))
    
    # Derivative term
    derivative = (error - last_error) / dt
    last_error = error
    
    # Calculate output (duty cycle)
    output = (Kp * error) + (Ki * integral) + (Kd * derivative)
    
    # Drive motor
    set_motor(output)
    
    # 3. Update OLED Display
    oled.fill(0)
    oled.rect(0, 0, 128, 14, 1)
    oled.text("JOINT PID HUD", 12, 3, 0)
    
    oled.text("Target: {} %".format(target_pos), 10, 20, 1)
    oled.text("Actual: {} %".format(actual_pos), 10, 32, 1)
    oled.text("Error : {}".format(int(error)), 10, 44, 1)
    oled.text("Output: {:.1f} %".format(output), 10, 56, 1)
    
    oled.rect(0, 0, 128, 64, 1)
    oled.show()
    
    print("Tgt:{} | Act:{} | Err:{} | Out:{:.1f}".format(
        target_pos, actual_pos, int(error), output
    ))
    
    utime.sleep_ms(20)  # Wait 20 ms (50 Hz loop frequency)
```

## What to Click in MbedO
1. Drag **Pico**, **two Potentiometers**, **L298N**, **DC Motor**, and **SSD1306 OLED** onto the canvas.
2. Connect Feedback Pot to **GP26**, Target Pot to **GP27**, L298N to **GP10-GP12**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the Target potentiometer (GP27). Verify that the motor spins to drive the Feedback potentiometer (GP26) to match the setpoint, automatically slowing down as the error approaches zero.

## Expected Output
Terminal:
```
Joint PID position controller active.
Tgt:50 | Act:20 | Err:30 | Out:32.4
Tgt:50 | Act:45 | Err:5  | Out:8.2
Tgt:50 | Act:50 | Err:0  | Out:0.0
```

## Expected Canvas Behavior
* Startup: OLED reads `JOINT PID HUD`.
* Adjust Target Pot (GP27): Motor spins. The value of `Actual` changes until it matches the `Target` value.
* Output duty cycle decreases as `Actual` gets closer to `Target`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `error = target_pos - actual_pos` | Calculates the current position error (difference between setpoint and feedback). |
| `integral = max(-100, min(100, ...))` | Accumulates error over time, clamped to prevent integral windup and motor overshoot. |

## Hardware & Safety Concept: Closed-Loop vs Open-Loop Control
Open-loop control systems command an output (like spinning a motor for 2 seconds) and assume the action was completed. Any mechanical load, wear, or battery drop will corrupt the final position. Closed-loop PID control systems continuously read sensor feedback (actual position), compare it to the target, and actively adjust the motor speed and direction in real-time, correcting for any external disturbances or resistance.

## Try This! (Challenges)
1. **Target Sweep Trajectory**: Write a loop that automatically sweeps the target position back and forth from 10% to 90% every 5 seconds, making the motor track the path automatically.
2. **Alert indicator**: Sound a buzzer on GP16 if the error remains greater than 15% for more than 3 seconds (indicating a mechanical joint jam).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor spins continuously in one direction | Positive feedback loop | The motor and feedback connections are inverted. Swap the L298N motor output wires or invert the sign of the error calculation in code. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [142 - Pico NTC Thermistor PID Temperature Controller](142-pico-ntc-thermistor-pid-temperature-controller.md)
- [159 - Pico DC Motor Encoder Closed-Loop Speed Control](159-pico-dc-motor-encoder-closed-loop-speed-control.md)
