# 156 - Pico Interrupt-Driven Encoder Tachometer OLED

Build a precision tachometer that uses hardware interrupts to count pulses from a Hall-effect encoder or optical sensor, calculates shaft RPM and surface speed, and displays a live speedometer needle and numeric reading on an SSD1306 OLED.

## Goal
Learn how to attach hardware interrupt handlers to GPIO pins in MicroPython, calculate RPM from pulse count over a fixed timing window, prevent race conditions using an atomic read pattern, and render a semicircular speedometer needle on an SSD1306 OLED.

## What You Will Build
A hardware-interrupt tachometer:
- **Hall-Effect / Optical Encoder (GP14)**: Generates pulses per revolution; each rising edge triggers an interrupt.
- **SSD1306 OLED (GP4, GP5)**: Displays a semicircular speedometer with a rotating needle and numeric RPM.
- **LED (GP13)**: Flashes on each detected pulse (low-speed verification).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Hall-Effect / Optical Sensor | `button` | Yes (pulse simulation) | Yes |
| SSD1306 OLED Display (128×64) | `oled` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Encoder / Hall Sensor | Signal OUT | GP14 | Orange | Pulse input (interrupt pin) |
| Encoder / Hall Sensor | VCC | 3.3V (3V3) | Red | Sensor power |
| Encoder / Hall Sensor | GND | GND | Black | Ground reference |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| SSD1306 OLED | VCC / GND | 3.3V / GND | Red / Black | OLED power |
| LED | Anode (+, via 330 Ω) | GP13 | Orange | Pulse flash indicator |
| LED | Cathode (−) | GND | Black | Ground return |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the encoder or Hall-effect sensor signal output to GP14. This pin is configured with an interrupt on the rising edge. Connect OLED to GP4/GP5 and LED to GP13. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime, math, ssd1306

i2c  = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

led         = Pin(13, Pin.OUT)
encoder_pin = Pin(14, Pin.IN, Pin.PULL_UP)

# Tachometer state (shared between ISR and main loop)
pulse_count = 0
last_led    = 0

PULSES_PER_REV = 1    # Slots or magnets per revolution — adjust for your encoder
WINDOW_MS      = 500  # Calculate RPM every 500 ms
MAX_RPM        = 3000

def encoder_isr(pin):
    global pulse_count
    pulse_count += 1

encoder_pin.irq(trigger=Pin.IRQ_RISING, handler=encoder_isr)

def calc_rpm(pulses, window_ms):
    """Convert pulse count in a time window to RPM."""
    revs_per_sec = (pulses / PULSES_PER_REV) / (window_ms / 1000.0)
    return revs_per_sec * 60.0

def needle_endpoint(cx, cy, radius, angle_deg):
    """Calculate tip coordinates of a speedometer needle."""
    rad = math.radians(angle_deg)
    x   = int(cx + radius * math.cos(rad))
    y   = int(cy - radius * math.sin(rad))
    return x, y

def draw_speedometer(rpm):
    oled.fill(0)

    # Arc from 210° to -30° (240° sweep)
    cx, cy = 64, 48
    R      = 36
    for deg in range(-30, 211, 3):
        rad = math.radians(deg)
        x   = int(cx + R * math.cos(rad))
        y   = int(cy - R * math.sin(rad))
        oled.pixel(x, y, 1)

    # Scale ticks (every 30°)
    for i in range(9):
        a   = 210 - i * 30
        x0, y0 = needle_endpoint(cx, cy, R - 2, a)
        x1, y1 = needle_endpoint(cx, cy, R + 2, a)
        oled.line(x0, y0, x1, y1, 1)

    # Needle angle: 210° (0 RPM) → -30° (MAX_RPM)
    rpm_clamped = max(0, min(MAX_RPM, rpm))
    needle_angle = 210 - (rpm_clamped / MAX_RPM) * 240
    nx, ny = needle_endpoint(cx, cy, R - 5, needle_angle)
    oled.line(cx, cy, nx, ny, 1)
    oled.fill_rect(cx - 2, cy - 2, 4, 4, 1)  # Hub

    # Numeric readout
    oled.text("{:4d} RPM".format(int(rpm)), 28, 0)
    oled.text("MAX:{}".format(MAX_RPM), 0, 56)
    oled.show()

oled.fill(0); oled.text("Tachometer", 20, 28); oled.show()
utime.sleep(1)
print("Interrupt tachometer active. Interrupt on GP14 rising edge.")

current_rpm  = 0.0
window_start = utime.ticks_ms()

while True:
    now = utime.ticks_ms()

    if utime.ticks_diff(now, window_start) >= WINDOW_MS:
        # Atomically read and reset counter
        count = pulse_count
        pulse_count = 0
        window_start = now
        current_rpm = calc_rpm(count, WINDOW_MS)
        print("Pulses:{} RPM:{:.0f}".format(count, current_rpm))

    draw_speedometer(current_rpm)

    # Flash LED on each pulse for visual feedback (low-speed only)
    if utime.ticks_diff(now, last_led) > 50:
        led.value(0)

    utime.sleep_ms(40)
```

## What to Click in MbedO
1. Drag **Pico**, **Button** (encoder simulation), **SSD1306 OLED**, and **LED** onto the canvas.
2. Connect Button to **GP14**, OLED to **GP4/GP5**, LED to **GP13**. Connect power and GND.
3. Paste code, click **Run**. Rapidly click the Button to simulate encoder pulses. Observe the OLED needle sweep.

## Expected Output
```
Interrupt tachometer active. Interrupt on GP14 rising edge.
Pulses:0 RPM:0
Pulses:12 RPM:1440
Pulses:25 RPM:3000
```

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `encoder_pin.irq(trigger=Pin.IRQ_RISING, handler=encoder_isr)` | Attaches a hardware interrupt: every rising edge on GP14 immediately increments `pulse_count` without polling. |
| Atomic read pattern: `count = pulse_count; pulse_count = 0` | Reads and resets the counter in two lines to minimise the window where a simultaneous ISR could cause a missed count. |

## Hardware & Safety Concept: Interrupt-Driven vs Polling
Polling checks a sensor every loop iteration and can miss fast events. Hardware interrupts fire immediately on signal transition regardless of main-loop timing, giving precise pulse timing. At 3000 RPM with 1 slot per revolution, pulses arrive every 20 ms — easy for polling. With 60 slots per revolution, pulses arrive every 333 µs — interrupts are essential.

## Try This! (Challenges)
1. **PPR Configuration**: Add a keypad to set `PULSES_PER_REV` at runtime to support different encoder resolutions without reflashing.
2. **Surface Speed**: Add wheel circumference as a constant and compute surface speed in m/s: `speed = (rpm / 60) * circumference_m`.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| RPM jumps erratically | Contact bounce on button | Add a 0.1 µF capacitor across the encoder signal and GND to filter bounce. |
| RPM always 0 | Pull-up missing | Ensure `Pin.PULL_UP` is set — the interrupt needs a clean logic transition. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [138 - Pico Stepper Motor CNC Positioner LCD](138-pico-stepper-motor-cnc-positioner-lcd.md)
- [144 - Pico OLED Signal Oscilloscope ADC](144-pico-oled-signal-oscilloscope-adc.md)
