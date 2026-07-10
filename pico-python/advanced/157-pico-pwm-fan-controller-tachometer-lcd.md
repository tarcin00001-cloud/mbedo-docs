# 157 - Pico PWM Fan Controller Tachometer LCD

Build a closed-loop PWM fan controller that reads fan speed via a tachometer signal, adjusts fan PWM duty to maintain a target RPM setpoint, and displays actual RPM, target RPM, and duty cycle on an I2C LCD.

## Goal
Learn how to implement closed-loop PWM fan speed control using interrupt-based tachometer feedback, build a proportional speed controller, and display the control state on an I2C character LCD in MicroPython.

## What You Will Build
A closed-loop fan speed controller:
- **4-Pin PWM Fan — Tach (GP14)**: Tachometer pulse input (2 pulses per revolution).
- **4-Pin PWM Fan — PWM Control (GP10)**: Fan speed control output (25 kHz PWM).
- **Potentiometer (GP26)**: Sets target RPM setpoint.
- **I2C 16x2 LCD (GP4, GP5)**: Displays actual RPM, target RPM, and duty %.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| PC 4-Pin PWM Fan | `dc_motor` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 10 kΩ Pull-up Resistor | `resistor` | Optional in MbedO | Yes (tach line pull-up) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| 4-Pin Fan | Pin 1 — GND | GND | Black | Fan ground |
| 4-Pin Fan | Pin 2 — +12V | External 12V+ | Red | Fan power (12V DC) |
| 4-Pin Fan | Pin 3 — Tach | GP14 | Yellow | Tachometer pulse (2 pulses/rev) |
| 4-Pin Fan | Pin 4 — PWM | GP10 | Orange | PWM control signal (25 kHz) |
| 10 kΩ Resistor | Either leg | GP14 to 3.3V | — | Tach pull-up (open-drain output) |
| Potentiometer | Wiper | GP26 | Yellow | Target RPM setpoint |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The fan tachometer is open-drain — it requires a pull-up resistor (10 kΩ to 3.3V) on the tach line. The 12V fan power must come from an external supply; connect the fan GND to the Pico GND. Connect OLED to GP4/GP5 and potentiometer to GP26.

## Code
```python
from machine import Pin, PWM, ADC, I2C
import utime
from machine_lcd import I2cLcd

# Fan PWM output at 25 kHz (Intel 4-pin standard)
fan_pwm = PWM(Pin(10))
fan_pwm.freq(25000)
fan_pwm.duty_u16(0)

# Tachometer interrupt
tach_pin  = Pin(14, Pin.IN, Pin.PULL_UP)
tach_count = 0

def tach_isr(pin):
    global tach_count
    tach_count += 1

tach_pin.irq(trigger=Pin.IRQ_FALLING, handler=tach_isr)

pot = ADC(26)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Controller parameters
PULSES_PER_REV = 2         # Standard PC fans: 2 tach pulses per revolution
MIN_RPM        = 300
MAX_RPM        = 2400
WINDOW_MS      = 1000      # Measure period

# Proportional gain
Kp = 0.03  # duty fraction per RPM error

duty_pct      = 50.0
window_start  = utime.ticks_ms()
actual_rpm    = 0.0

lcd.clear(); lcd.putstr("Fan Controller"); utime.sleep(1)
print("PWM fan controller active.")

while True:
    now = utime.ticks_ms()

    # Read target from potentiometer
    raw        = pot.read_u16()
    target_rpm = MIN_RPM + int(raw * (MAX_RPM - MIN_RPM) / 65535)

    if utime.ticks_diff(now, window_start) >= WINDOW_MS:
        # Calculate RPM
        count      = tach_count
        tach_count = 0
        window_start = now

        revs       = count / PULSES_PER_REV
        actual_rpm = revs * 60.0  # Per minute

        # Proportional control
        error    = target_rpm - actual_rpm
        duty_pct = max(0.0, min(100.0, duty_pct + Kp * error))

        fan_pwm.duty_u16(int(duty_pct * 65535 / 100))

        print("Target:{} Actual:{:.0f} Duty:{:.0f}%".format(
            target_rpm, actual_rpm, duty_pct))

    # LCD
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("Set:{:4d} Act:{:4d}".format(target_rpm, int(actual_rpm)))
    lcd.move_to(0, 1)
    lcd.putstr("Duty: {:3.0f}%  RPM  ".format(duty_pct))

    utime.sleep_ms(100)
```

## What to Click in MbedO
1. Drag **Pico**, **DC Motor** (fan), **Potentiometer**, and **I2C LCD** onto the canvas.
2. Connect Tach signal (fan output) to **GP14**, PWM control to **GP10**, Potentiometer to **GP26**, LCD to **GP4/GP5**. Connect power and GND.
3. Paste code, click **Run**. Slide the potentiometer to set target RPM. Observe the duty adjust to reach the setpoint.

## Expected Output
```
PWM fan controller active.
Target:1200 Actual:0 Duty:50%
Target:1200 Actual:850 Duty:68%
Target:1200 Actual:1185 Duty:70%
```

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `fan_pwm.freq(25000)` | Sets the Intel-standard 25 kHz PWM frequency for 4-pin PC fans — fans ignore lower frequencies and run at full speed. |
| `duty_pct += Kp * error` | Proportional control: if actual RPM is below target, increase duty; if above, decrease. |

## Hardware & Safety Concept: 4-Pin Fan PWM Standard
Intel's CPU fan specification defines: Pin 1 = GND, Pin 2 = 12V power, Pin 3 = Tachometer (open-drain, 2 pulses/rev), Pin 4 = PWM control at 25 kHz. The PWM signal controls speed without varying supply voltage — eliminating the motor heating that occurs with lower-frequency voltage-chopped control.

## Try This! (Challenges)
1. **PI Controller**: Add an integral term (`integral += error * dt; duty += Ki * integral`) to eliminate the steady-state RPM offset that the proportional-only controller leaves.
2. **Thermal Control**: Replace the potentiometer with a DHT22 sensor and automatically set target RPM based on measured temperature.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Fan runs at full speed regardless of PWM | Frequency not 25 kHz | Verify `fan_pwm.freq(25000)` — 3-pin fans ignore the PWM pin entirely. |
| Tach always reads 0 | Missing pull-up | Connect a 10 kΩ resistor from tach pin to 3.3V. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [156 - Pico Interrupt-Driven Encoder Tachometer OLED](156-pico-interrupt-driven-encoder-tachometer-oled.md)
- [114 - Pico Smart Fan DHT22 DC Motor LCD](../intermediate/114-pico-smart-fan-dht22-dc-motor-lcd.md)
