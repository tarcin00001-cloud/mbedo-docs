# 144 - Pico OLED Signal Oscilloscope ADC

Build a simple single-channel oscilloscope that samples an analog signal on an ADC pin, stores a 128-sample frame, and renders a real-time waveform plot on an SSD1306 OLED display with voltage scale and frequency estimate.

## Goal
Learn how to acquire a burst of ADC samples at a controlled rate, scale the values to OLED pixel coordinates, render a polyline waveform, and calculate a basic zero-crossing frequency estimate for display in MicroPython.

## What You Will Build
A single-channel analog oscilloscope:
- **Analog Signal Input (GP26)**: The signal under observation (connect a function generator, audio output, or potentiometer).
- **SSD1306 OLED (GP4, GP5)**: Renders the live waveform and displays voltage scale and frequency.
- **Button (GP14)**: Freezes/unfreezes the display (trigger hold).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Potentiometer / Signal Source | `potentiometer` | Yes | Yes |
| SSD1306 OLED Display (128×64) | `oled` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Signal Source / Potentiometer | Output / Wiper | GP26 | Yellow | ADC analog input (max 3.3V) |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| SSD1306 OLED | VCC / GND | 3.3V / GND | Red / Black | OLED power |
| Button | Terminal 1 / 2 | GP14 / GND | White / Black | Freeze/unfreeze trigger |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The signal to be observed connects to GP26 (ADC channel 0). Maximum input is 3.3V — never connect signals exceeding this voltage directly to the Pico GPIO. Connect the OLED to GP4/GP5. Connect the Freeze button to GP14. All grounds are shared.

## Code
```python
from machine import Pin, ADC, I2C
import utime
import ssd1306

adc    = ADC(26)
freeze_btn = Pin(14, Pin.IN, Pin.PULL_UP)

i2c  = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

# Display layout constants
PLOT_X   = 0
PLOT_Y   = 10   # Y start of plot area
PLOT_W   = 128
PLOT_H   = 48   # Plot area height (pixels)
SAMPLES  = 128

# Sample interval (microseconds) — adjust for different time scales
SAMPLE_US = 500  # 500µs per sample = ~2 kHz Nyquist

frozen   = False
last_btn = 0
frame    = [32767] * SAMPLES   # Last rendered frame

def acquire():
    buf = []
    for _ in range(SAMPLES):
        buf.append(adc.read_u16())
        utime.sleep_us(SAMPLE_US)
    return buf

def count_zero_crossings(buf):
    mid   = 32767
    cross = 0
    prev  = buf[0] >= mid
    for v in buf[1:]:
        curr = v >= mid
        if curr != prev:
            cross += 1
        prev = curr
    # Frequency = (crossings / 2) / total_time
    total_time_s = SAMPLES * SAMPLE_US / 1e6
    return (cross / 2) / total_time_s if total_time_s > 0 else 0

def draw_frame(buf):
    oled.fill(0)
    # Header
    v_mid   = buf[len(buf)//2] * 3.3 / 65535
    freq_hz = count_zero_crossings(buf)
    oled.text("{:.2f}V {}Hz".format(v_mid, int(freq_hz)), 0, 0)
    oled.hline(0, PLOT_Y - 1, PLOT_W, 1)

    # Midpoint reference line
    oled.hline(0, PLOT_Y + PLOT_H // 2, PLOT_W, 1)

    # Waveform plot
    prev_py = None
    for x in range(SAMPLES):
        val  = buf[x]
        # Map 0-65535 ADC to PLOT_H pixels (inverted Y)
        py   = PLOT_Y + PLOT_H - 1 - int(val * (PLOT_H - 1) / 65535)
        if prev_py is not None:
            oled.line(x - 1, prev_py, x, py, 1)
        prev_py = py

    # Freeze indicator
    if frozen:
        oled.text("HOLD", 100, 0)
    oled.show()

oled.fill(0); oled.text("Oscilloscope", 16, 24); oled.show()
utime.sleep(1)
print("OLED oscilloscope active. Press button to freeze.")

while True:
    now = utime.ticks_ms()

    # Freeze button toggle
    if freeze_btn.value() == 0 and utime.ticks_diff(now, last_btn) > 400:
        frozen   = not frozen
        last_btn = now
        print("Frozen:", frozen)

    if not frozen:
        frame = acquire()

    draw_frame(frame)
```

## What to Click in MbedO
1. Drag **Pico**, **Potentiometer**, **SSD1306 OLED**, and **Push Button** onto the canvas.
2. Connect Potentiometer to **GP26**, OLED to **GP4/GP5**, Button to **GP14**. Connect power and GND.
3. Paste code, click **Run**. Slowly move the potentiometer to observe the waveform changing on the OLED.
4. Click the Button to freeze the current waveform frame.

## Expected Output
```
OLED oscilloscope active. Press button to freeze.
Frozen: True
Frozen: False
```
(On OLED: a polyline waveform rendered across the full 128-pixel width with a voltage reading and frequency estimate in the header.)

## Expected Canvas Behavior
- Moving the potentiometer slider produces a slowly rising or falling waveform on the OLED. Clicking the Button stops updating the display (HOLD indicator appears in top-right corner).

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `utime.sleep_us(SAMPLE_US)` | Controls sample rate — 500µs spacing gives a 2 kHz Nyquist frequency. Reduce for higher frequencies. |
| `oled.line(x-1, prev_py, x, py, 1)` | Draws connected line segments between consecutive sample points, forming a continuous waveform. |
| `count_zero_crossings(buf)` | Estimates signal frequency by counting the number of times the signal crosses the midpoint per second. |

## Hardware & Safety Concept: Nyquist Sampling Theorem
The Nyquist theorem states the sample rate must be at least **twice the signal frequency** to reconstruct it faithfully. At 500µs per sample (2000 samples/second), the maximum observable frequency is 1000 Hz. Reduce `SAMPLE_US` to observe higher-frequency signals, noting that MicroPython loop overhead sets a practical floor of approximately 50µs per sample.

## Try This! (Challenges)
1. **Time Scale Button**: Add a second button that cycles `SAMPLE_US` between 100µs, 500µs, 1ms, and 5ms, displaying the time/division on screen.
2. **Dual-Channel**: Add a second ADC input (GP27) and display both channels overlaid on the OLED in different styles.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Waveform appears flat | Signal voltage out of ADC range | Ensure input signal is between 0V and 3.3V. Never connect higher voltages directly. |
| Waveform flickers rapidly | `SAMPLE_US` too small | Increase `SAMPLE_US` to 1000µs or more for stable display with slow signals. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [109 - Pico Ultrasonic Distance OLED](../intermediate/109-pico-ultrasonic-distance-oled.md)
- [139 - Pico MPU-6050 Digital Inclinometer OLED](139-pico-mpu6050-digital-inclinometer-oled.md)
