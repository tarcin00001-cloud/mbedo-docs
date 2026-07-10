# 159 - Pico DC Motor Encoder Closed-Loop Speed Control

Build a closed-loop DC motor speed controller that reads a quadrature encoder via interrupts, implements a PI controller to maintain target RPM, and displays actual speed, target speed, and error on an I2C LCD.

## Goal
Learn how to decode a quadrature encoder using two interrupt pins to get signed position and velocity, implement a PI controller for motor speed regulation, and drive the motor via an L298N while displaying the control system state on an I2C LCD in MicroPython.

## What You Will Build
A closed-loop DC motor speed controller:
- **Quadrature Encoder (GP14 A, GP15 B)**: Dual-phase encoder giving direction and speed.
- **L298N Motor Driver (GP10 ENA PWM, GP11 IN1, GP12 IN2)**: Drives the DC motor.
- **Potentiometer (GP26)**: Sets target RPM setpoint.
- **I2C 16x2 LCD (GP4, GP5)**: Displays actual RPM, target RPM, duty %, and error.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DC Motor with Quadrature Encoder | `dc_motor` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Encoder | Channel A | GP14 | Orange | Phase A interrupt input |
| Encoder | Channel B | GP15 | Yellow | Phase B direction input |
| Encoder | VCC / GND | 3.3V / GND | Red / Black | Encoder power |
| L298N | ENA (PWM) | GP10 | Orange | Motor speed (PWM) |
| L298N | IN1 | GP11 | Yellow | Motor direction A |
| L298N | IN2 | GP12 | Green | Motor direction B |
| L298N | Motor power | 6-12V external | Red | L298N motor supply |
| L298N | GND | GND | Black | Shared ground |
| Potentiometer | Wiper | GP26 | Yellow | Target RPM setpoint |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Both encoder channels (A on GP14, B on GP15) require interrupts. L298N ENA on GP10 (PWM), IN1/IN2 on GP11/GP12. Connect L298N motor power to an external 6V-12V supply. All grounds are shared.

## Code
```python
from machine import Pin, PWM, ADC, I2C
import utime
from machine_lcd import I2cLcd

# Encoder
enc_a    = Pin(14, Pin.IN, Pin.PULL_UP)
enc_b    = Pin(15, Pin.IN, Pin.PULL_UP)
enc_pos  = 0  # Signed position counter

# L298N
ena = PWM(Pin(10)); ena.freq(1000)
in1 = Pin(11, Pin.OUT); in2 = Pin(12, Pin.OUT)
in1.value(1); in2.value(0)  # Forward direction

pot = ADC(26)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

ENCODER_PPR = 20     # Pulses per revolution (encoder spec)
WINDOW_MS   = 200    # Speed calculation window
MIN_RPM     = 0
MAX_RPM     = 500

def encoder_isr(pin):
    global enc_pos
    if enc_b.value() == 0:
        enc_pos += 1
    else:
        enc_pos -= 1

enc_a.irq(trigger=Pin.IRQ_RISING, handler=encoder_isr)

# PI state
integral    = 0.0
duty_pct    = 30.0
Kp, Ki      = 0.05, 0.001
dt          = WINDOW_MS / 1000.0

prev_pos    = 0
actual_rpm  = 0.0
window_start = utime.ticks_ms()

def set_motor(duty):
    """Set motor speed: duty 0-100 = forward speed."""
    if duty >= 0:
        in1.value(1); in2.value(0)
    else:
        in1.value(0); in2.value(1)
    ena.duty_u16(min(65535, int(abs(duty) * 655)))

lcd.clear(); lcd.putstr("Motor Speed Ctl"); utime.sleep(1)
print("DC motor closed-loop controller active.")

while True:
    now = utime.ticks_ms()

    # Read target
    raw        = pot.read_u16()
    target_rpm = MIN_RPM + int(raw * (MAX_RPM - MIN_RPM) / 65535)

    if utime.ticks_diff(now, window_start) >= WINDOW_MS:
        # Calculate speed from encoder position change
        delta    = enc_pos - prev_pos
        prev_pos = enc_pos
        window_start = now

        revs       = delta / ENCODER_PPR
        actual_rpm = abs(revs / dt * 60)

        # PI controller
        error      = target_rpm - actual_rpm
        integral   = max(-500, min(500, integral + error * dt))
        duty_pct   = max(0, min(100, Kp * error + Ki * integral +
                       (30 if target_rpm > 0 else 0)))
        set_motor(duty_pct)

        print("Tgt:{} Act:{:.0f} Err:{:.0f} Duty:{:.0f}%".format(
            target_rpm, actual_rpm, error, duty_pct))

    # LCD update
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("Tgt:{:4d} Act:{:4d}".format(target_rpm, int(actual_rpm)))
    lcd.move_to(0, 1)
    lcd.putstr("Duty:{:3.0f}% Pos:{:5d}".format(duty_pct, enc_pos % 10000))

    utime.sleep_ms(50)
```

## What to Click in MbedO
1. Drag **Pico**, **DC Motor with Encoder**, **L298N**, **Potentiometer**, and **I2C LCD** onto the canvas.
2. Connect Encoder A to **GP14**, B to **GP15**. Connect L298N ENA to **GP10**, IN1/IN2 to **GP11/GP12**. Connect Potentiometer to **GP26**. Connect LCD to **GP4/GP5**. Connect power and GND.
3. Paste code, click **Run**. Slide the potentiometer to set the target RPM. Observe the PI controller adjusting motor duty.

## Expected Output
```
DC motor closed-loop controller active.
Tgt:300 Act:0 Err:300 Duty:45%
Tgt:300 Act:180 Err:120 Duty:60%
Tgt:300 Act:295 Err:5 Duty:62%
```

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `enc_b.value() == 0` in ISR | Direction decoding: reads phase B state at each phase A rising edge to determine CW (+1) or CCW (−1) motion. |
| `integral = max(-500, min(500, ...))` | Anti-windup clamp: prevents the integral term accumulating unboundedly when the motor cannot reach the setpoint. |
| Feed-forward term `(30 if target_rpm > 0 else 0)` | Provides an initial duty bias so the motor overcomes static friction before the PI action builds up. |

## Hardware & Safety Concept: Feed-Forward + Feedback Control
Pure feedback control (PI/PID) reacts to error after it occurs. Adding a feed-forward term (a constant duty when target > 0) pre-loads the output to approximately the right level, reducing the initial error the integrator must compensate. This hybrid approach is standard in industrial servo drives for faster setpoint response.

## Try This! (Challenges)
1. **Velocity Profiling**: Instead of jumping to target RPM immediately, ramp the setpoint linearly at a configurable acceleration rate (RPM/second) to prevent mechanical shock.
2. **Position Mode**: Add a second mode where `enc_pos` is driven to a commanded position rather than speed, using P-only position control.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor runs but RPM always 0 | Encoder not connected | Verify GP14/GP15 are connected to encoder A/B channels with pull-ups. |
| Motor oscillates at setpoint | Kp too high | Reduce `Kp` from 0.05 to 0.02 for a stiffer, damped response. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [156 - Pico Interrupt-Driven Encoder Tachometer OLED](156-pico-interrupt-driven-encoder-tachometer-oled.md)
- [157 - Pico PWM Fan Controller Tachometer LCD](157-pico-pwm-fan-controller-tachometer-lcd.md)
