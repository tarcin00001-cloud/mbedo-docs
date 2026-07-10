# 147 - Pico Sound Level Meter Microphone OLED

Build a sound level meter that reads a MAX4466 electret microphone amplifier, calculates the RMS (root-mean-square) amplitude from a burst of ADC samples, displays a live bar graph on an SSD1306 OLED, and triggers a visual peak indicator.

## Goal
Learn how to calculate RMS amplitude from burst ADC samples, convert the result to decibels (dBFS), render a dynamic level bar graph on an SSD1306 OLED, and detect transient peaks with a hold timer in MicroPython.

## What You Will Build
A sound level meter:
- **MAX4466 Microphone (GP26)**: Reads audio amplitude as an analog voltage.
- **SSD1306 OLED (GP4, GP5)**: Displays a bar-graph level meter and dBFS reading.
- **LED (GP13)**: Lights when sound level exceeds the clip threshold.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MAX4466 Microphone Amplifier | `potentiometer` | Yes (slider simulates amplitude) | Yes |
| SSD1306 OLED Display (128×64) | `oled` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MAX4466 | VCC | 3.3V (3V3) | Red | Module power |
| MAX4466 | GND | GND | Black | Ground reference |
| MAX4466 | OUT | GP26 | Yellow | Amplified microphone signal |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| SSD1306 OLED | VCC / GND | 3.3V / GND | Red / Black | OLED power |
| LED | Anode (+, via 330 Ω) | GP13 | Red | Clip/overload indicator |
| LED | Cathode (−) | GND | Black | Ground return |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the MAX4466 OUT to GP26. The module amplifies microphone audio to 0–3.3V with quiescent DC at ~1.65V (mid-rail). Connect OLED to GP4/GP5 and LED to GP13.

## Code
```python
from machine import Pin, ADC, I2C
import utime, math, ssd1306

mic  = ADC(26)
led  = Pin(13, Pin.OUT)
i2c  = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

led.value(0)

SAMPLES      = 256
DC_OFFSET    = 32767    # Mid-rail DC bias for AC-coupled audio
CLIP_DBFS    = -6.0     # Threshold for clip LED
PEAK_HOLD_MS = 1000     # Peak hold time

peak_dbfs    = -60.0
peak_hold_ts = utime.ticks_ms()

def measure_rms():
    total = 0
    for _ in range(SAMPLES):
        raw   = mic.read_u16()
        v     = raw - DC_OFFSET
        total += v * v
    rms = math.sqrt(total / SAMPLES)
    return rms

def rms_to_dbfs(rms):
    if rms < 1:
        return -60.0
    return 20.0 * math.log10(rms / 32767)

def draw_meter(dbfs, peak):
    oled.fill(0)
    oled.text("SOUND LEVEL", 16, 0)
    oled.hline(0, 10, 128, 1)

    # Numeric reading
    oled.text("{:.1f} dBFS".format(dbfs), 0, 14)

    # Level bar: map -60 dBFS to 0 dBFS → 0 to 112 pixels
    bar_len = max(0, min(112, int((dbfs + 60) * 112 / 60)))
    oled.rect(8, 28, 112, 14, 1)

    # Colour zones: green 0-70%, orange 70-90%, red 90-100%
    g_end = int(112 * 0.7)
    o_end = int(112 * 0.9)
    if bar_len > 0:
        oled.fill_rect(8, 28, min(bar_len, g_end), 14, 1)
    if bar_len > g_end:
        oled.fill_rect(8 + g_end, 28, min(bar_len - g_end, o_end - g_end), 14, 1)
    if bar_len > o_end:
        oled.fill_rect(8 + o_end, 28, bar_len - o_end, 14, 1)

    # Peak hold marker
    peak_x = max(8, min(119, 8 + int((peak + 60) * 112 / 60)))
    oled.vline(peak_x, 28, 14, 1)

    # Labels
    oled.text("-60", 0, 44)
    oled.text("-30", 44, 44)
    oled.text("0", 116, 44)
    oled.text("Peak:{:.1f}".format(peak), 0, 54)
    oled.show()

oled.fill(0); oled.text("Sound Meter", 16, 24); oled.show()
utime.sleep(1)
print("Sound level meter active.")

while True:
    rms   = measure_rms()
    dbfs  = rms_to_dbfs(rms)
    now   = utime.ticks_ms()

    # Update peak hold
    if dbfs > peak_dbfs or utime.ticks_diff(now, peak_hold_ts) > PEAK_HOLD_MS:
        if dbfs > peak_dbfs:
            peak_dbfs    = dbfs
            peak_hold_ts = now
        else:
            peak_dbfs    = dbfs  # Hold expired — reset

    # Clip LED
    led.value(1 if dbfs > CLIP_DBFS else 0)

    draw_meter(dbfs, peak_dbfs)
    print("RMS:{:.0f} dBFS:{:.1f} Peak:{:.1f}".format(rms, dbfs, peak_dbfs))
```

## What to Click in MbedO
1. Drag **Pico**, **Potentiometer** (microphone), **SSD1306 OLED**, and **LED** onto the canvas.
2. Connect Potentiometer to **GP26**, OLED to **GP4/GP5**, LED to **GP13**. Connect power and GND.
3. Paste code, click **Run**. Slide the potentiometer to simulate different volume levels.
4. Slide above the clip threshold (near maximum) to activate the clip LED.

## Expected Output
```
Sound level meter active.
RMS:820 dBFS:-32.0 Peak:-28.5
RMS:3200 dBFS:-20.2 Peak:-20.2
```
(On OLED: a horizontal bar grows and a vertical peak-hold marker sits at the highest recent level.)

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `total += v * v` then `math.sqrt(total / SAMPLES)` | RMS calculation: sum of squares divided by count, then square root gives the true power-equivalent amplitude. |
| `20.0 * math.log10(rms / 32767)` | Converts RMS amplitude to dBFS (decibels relative to full scale), where 0 dBFS is the maximum digital value. |
| Peak hold logic | The peak is held for `PEAK_HOLD_MS` ms and then resets, matching the VU meter standard behaviour. |

## Hardware & Safety Concept: RMS vs Peak Amplitude
Peak amplitude measures the instantaneous maximum value. RMS (Root Mean Square) measures the power-equivalent average — it equals the DC voltage that would deliver the same power to a resistive load. For audio, RMS corresponds to perceived loudness far better than peak detection, which is why all professional audio meters display RMS levels.

## Try This! (Challenges)
1. **A-Weighting**: Apply a simplified A-weighting filter to the samples before RMS calculation to better match human perceived loudness.
2. **Clap Counter**: Detect rapid transient peaks above -20 dBFS separated by at least 200 ms and count them as clap events.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Meter always shows -60 dBFS | DC offset wrong | Print raw ADC values in silence and set `DC_OFFSET` to the measured quiet value. |
| Meter never reaches 0 dBFS | Mic gain too low | Adjust MAX4466 gain pot to maximum, or reduce `DC_OFFSET` if signal is not centred at mid-rail. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [144 - Pico OLED Signal Oscilloscope ADC](144-pico-oled-signal-oscilloscope-adc.md)
- [139 - Pico MPU-6050 Digital Inclinometer OLED](139-pico-mpu6050-digital-inclinometer-oled.md)
