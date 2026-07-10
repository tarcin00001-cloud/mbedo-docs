# 142 - Pico NTC Thermistor PID Temperature Controller

Build a closed-loop PID temperature controller that reads an NTC thermistor, calculates a proportional-integral-derivative correction, drives a heating relay at variable PWM duty, and displays setpoint, measured temperature, and PID output on an I2C LCD.

## Goal
Learn how to implement a software PID controller in MicroPython, read temperature from an NTC thermistor using the Steinhart-Hart equation, and output a PWM duty cycle to control a solid-state relay or heating element.

## What You Will Build
A PID temperature controller:
- **NTC Thermistor (GP26)**: Reads process temperature via resistor voltage divider.
- **PWM Heating Relay (GP10)**: Controlled at variable duty by the PID output.
- **Button Up (GP13)**: Increases target setpoint by 1°C.
- **Button Down (GP14)**: Decreases target setpoint by 1°C.
- **I2C 16x2 LCD (GP4, GP5)**: Displays setpoint, measured temp, and PID output percentage.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| NTC 10kΩ Thermistor | `potentiometer` | Yes (slider simulates temperature) | Yes |
| Relay Module (or SSR) | `relay` | Yes | Yes |
| Tactile Push Button × 2 | `button` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 10 kΩ Fixed Resistor | `resistor` | Optional in MbedO | Yes (thermistor voltage divider) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NTC Thermistor | Pin 1 | 3.3V (3V3) | Red | Top of voltage divider |
| NTC Thermistor | Pin 2 | GP26 | Yellow | ADC junction point |
| 10 kΩ Resistor | Either leg | GP26 to GND | — | Bottom of voltage divider |
| Relay / SSR | IN (PWM) | GP10 | Orange | Heating element control |
| Button Up | Terminal 1 / 2 | GP13 / GND | White / Black | Setpoint increase |
| Button Down | Terminal 1 / 2 | GP14 / GND | White / Black | Setpoint decrease |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The NTC thermistor and 10 kΩ fixed resistor form a voltage divider: 3.3V → NTC → GP26 → 10kΩ → GND. As temperature rises, NTC resistance drops, and GP26 voltage rises. Connect relay IN to GP10, buttons to GP13/GP14, and LCD to GP4/GP5.

## Code
```python
from machine import Pin, ADC, PWM, I2C
import utime, math
from machine_lcd import I2cLcd

ntc    = ADC(26)
relay  = PWM(Pin(10)); relay.freq(10)  # Slow PWM for relay duty
btn_up = Pin(13, Pin.IN, Pin.PULL_UP)
btn_dn = Pin(14, Pin.IN, Pin.PULL_UP)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Steinhart-Hart NTC parameters (B=3950, R0=10kΩ at 25°C)
B_COEFF = 3950.0
R0      = 10000.0
R_FIXED = 10000.0
T0_K    = 298.15  # 25°C in Kelvin

def ntc_temperature():
    raw  = ntc.read_u16()
    if raw == 0: raw = 1
    v    = raw * 3.3 / 65535
    r_ntc = R_FIXED * v / (3.3 - v)
    inv_t = (1.0 / T0_K) + (1.0 / B_COEFF) * math.log(r_ntc / R0)
    return (1.0 / inv_t) - 273.15

# PID configuration
Kp, Ki, Kd = 2.0, 0.5, 1.0
setpoint    = 35.0

# PID state
integral   = 0.0
prev_error = 0.0
dt         = 0.5   # Sample time in seconds

OUTPUT_MAX = 65535
OUTPUT_MIN = 0

relay.duty_u16(0)

SETPOINT_MIN = 20.0
SETPOINT_MAX = 80.0

last_btn = 0

lcd.clear(); lcd.putstr("PID Temp Ctrl"); utime.sleep(1)
print("PID temperature controller active.")

while True:
    now = utime.ticks_ms()

    # --- Setpoint buttons ---
    if btn_up.value() == 0 and utime.ticks_diff(now, last_btn) > 300:
        setpoint = min(SETPOINT_MAX, setpoint + 1.0); last_btn = now
        print("Setpoint: {:.0f}C".format(setpoint))
    if btn_dn.value() == 0 and utime.ticks_diff(now, last_btn) > 300:
        setpoint = max(SETPOINT_MIN, setpoint - 1.0); last_btn = now
        print("Setpoint: {:.0f}C".format(setpoint))

    # --- PID computation ---
    measured = ntc_temperature()
    error    = setpoint - measured

    integral    = max(-1000, min(1000, integral + error * dt))
    derivative  = (error - prev_error) / dt
    prev_error  = error

    pid_out = Kp * error + Ki * integral + Kd * derivative
    duty    = int(max(OUTPUT_MIN, min(OUTPUT_MAX, pid_out * (OUTPUT_MAX / 100.0))))
    out_pct = int(duty * 100 / OUTPUT_MAX)

    relay.duty_u16(duty)

    # --- LCD ---
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("SP:{:.0f} PV:{:.1f}C".format(setpoint, measured))
    lcd.move_to(0, 1)
    lcd.putstr("PID out: {:3d} %".format(out_pct))

    print("SP:{:.0f} PV:{:.1f} Err:{:.1f} Out:{:3d}%".format(
        setpoint, measured, error, out_pct))

    utime.sleep(dt)
```

## What to Click in MbedO
1. Drag **Pico**, **Potentiometer** (NTC thermistor), **Relay**, **two Buttons**, and **I2C LCD** onto the canvas.
2. Connect Potentiometer to **GP26**, Relay to **GP10**, Buttons to **GP13/GP14**, and LCD to **GP4/GP5**. Connect power and GND.
3. Paste code, click **Run**. Default setpoint is 35°C. Press Up/Down buttons to adjust.
4. Slide the NTC potentiometer to simulate temperature changes and observe the PID output adjust the relay duty.

## Expected Output
```
PID temperature controller active.
SP:35 PV:24.5 Err:10.5 Out: 21%
SP:35 PV:34.8 Err: 0.2 Out:  0%
Setpoint: 36C
```

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| Steinhart-Hart formula | Converts NTC resistance (from voltage divider ADC reading) to temperature in °C using the B-coefficient approximation. |
| `integral = max(-1000, min(1000, integral + error * dt))` | Anti-windup clamping: prevents the integral term from accumulating unboundedly when the output is saturated. |
| `Kp * error + Ki * integral + Kd * derivative` | Classic PID formula combining proportional (immediate), integral (steady-state), and derivative (predictive) terms. |

## Hardware & Safety Concept: PID Anti-Windup
Without anti-windup, if the process is far from setpoint (e.g. cold startup), the integral accumulates a large value. When temperature finally reaches setpoint, the large integral causes the controller to overshoot significantly before correcting. Clamping the integral term prevents this, producing a faster and more stable response.

## Try This! (Challenges)
1. **Auto-Tune**: Implement a simple relay-feedback auto-tuning method (Ziegler-Nichols) that measures system oscillation period to automatically estimate Kp, Ki, Kd.
2. **OLED Trend Plot**: Plot the last 30 seconds of measured temperature as a scrolling graph on an SSD1306 OLED.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Temperature reading always max | Thermistor not connected | Verify voltage divider circuit — open circuit at NTC shows full voltage on GP26. |
| Controller oscillates wildly | Kd too high | Reduce `Kd` to 0.1 and increase `dt` to 1 second for more stable derivative action. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [103 - Pico Temperature Alarm HVAC](../intermediate/103-pico-temperature-alarm-hvac.md)
- [123 - Pico Cold Chain Monitor DS18B20 Relay LCD](123-pico-cold-chain-monitor-ds18b20-relay-lcd.md)
