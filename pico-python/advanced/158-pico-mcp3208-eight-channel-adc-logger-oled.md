# 158 - Pico MCP3208 Eight-Channel ADC Logger OLED

Build an eight-channel analogue data acquisition system using an MCP3208 SPI ADC, scan all eight channels at configurable sample rates, display a scrolling multi-channel bar graph on an SSD1306 OLED, and output readings to the serial console in CSV format.

## Goal
Learn how to interface the MCP3208 12-bit eight-channel SPI ADC with the Pico, implement the SPI communication protocol manually, scan all channels in a round-robin loop, and render a compact multi-bar display on an SSD1306 OLED in MicroPython.

## What You Will Build
An eight-channel ADC data acquisition system:
- **MCP3208 SPI ADC (GP10-GP13)**: Reads eight independent analog inputs at 12-bit resolution.
- **Eight Signal Sources / Potentiometers**: Any analog signals 0–3.3V connected to CH0-CH7.
- **SSD1306 OLED (GP4, GP5)**: Displays all eight channel values as a bar graph.
- **Button (GP14)**: Cycles between bar graph, numeric table, and max/min views.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MCP3208 8-Channel ADC | `spi_adc` | Yes | Yes |
| Potentiometer × 8 (or signal sources) | `potentiometer` | Yes | Yes |
| SSD1306 OLED Display (128×64) | `oled` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MCP3208 | SCK | GP10 | Blue | SPI Clock |
| MCP3208 | MOSI (DIN) | GP11 | Orange | SPI Master Out |
| MCP3208 | MISO (DOUT) | GP12 | Yellow | SPI Master In |
| MCP3208 | CS (CE) | GP13 | White | SPI Chip Select |
| MCP3208 | VDD / VREF | 3.3V (3V3) | Red | Supply and reference voltage |
| MCP3208 | DGND / AGND | GND | Black | Both grounds connected |
| CH0–CH7 (inputs) | Potentiometer wipers | MCP3208 CH0-CH7 | Yellow wires | Analog inputs 0–7 |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | I2C Bus 0 |
| SSD1306 OLED | VCC / GND | 3.3V / GND | Red / Black | OLED power |
| Button | Terminal 1 / 2 | GP14 / GND | White / Black | View cycle button |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** MCP3208 uses SPI1 (GP10-GP13). VREF = VDD = 3.3V gives a full-scale range of 3.3V. All eight analog inputs connect to CH0-CH7. OLED uses I2C Bus 0 on GP4/GP5. Both DGND and AGND pins of MCP3208 must be grounded.

## Code
```python
from machine import Pin, SPI, I2C
import utime, ssd1306

# MCP3208 on SPI1
spi = SPI(1, sck=Pin(10), mosi=Pin(11), miso=Pin(12), baudrate=1000000)
cs  = Pin(13, Pin.OUT); cs.value(1)

# OLED on I2C0
i2c  = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

btn      = Pin(14, Pin.IN, Pin.PULL_UP)
last_btn = 0
view     = 0  # 0=bars, 1=numeric, 2=min/max
VIEW_NAMES = ["BAR GRAPH", "NUMERIC  ", "MIN / MAX"]

# Per-channel history
ch_min  = [4095] * 8
ch_max  = [0]    * 8

def mcp3208_read(channel):
    """Read single-ended channel 0-7 from MCP3208."""
    cmd = 0b00000110 | ((channel & 0b100) >> 2)  # Start + SGL + D2
    byte2 = ((channel & 0b011) << 6)              # D1 D0 + 0 padding
    cs.value(0)
    spi.write(bytes([cmd, byte2, 0x00]))
    cs.value(1)
    # Re-read with proper framing
    cs.value(0)
    buf = bytearray(3)
    spi.write_readinto(bytes([cmd, byte2, 0x00]), buf)
    cs.value(1)
    result = ((buf[1] & 0x0F) << 8) | buf[2]
    return result

def read_all():
    return [mcp3208_read(ch) for ch in range(8)]

def raw_to_v(raw):
    return raw * 3.3 / 4095

def draw_bars(readings):
    oled.fill(0)
    oled.text("8CH ADC LOGGER", 0, 0)
    oled.hline(0, 10, 128, 1)
    BAR_MAX_H = 46
    BAR_W     = 14
    for i, val in enumerate(readings):
        x       = 2 + i * 16
        bar_h   = max(1, int(val * BAR_MAX_H / 4095))
        oled.rect(x, 11, BAR_W, BAR_MAX_H, 1)
        oled.fill_rect(x, 11 + BAR_MAX_H - bar_h, BAR_W, bar_h, 1)
        oled.text(str(i), x + 4, 58)
    oled.show()

def draw_numeric(readings):
    oled.fill(0)
    oled.text("CH  RAW    V", 0, 0)
    oled.hline(0, 10, 128, 1)
    for i in range(4):
        v = raw_to_v(readings[i])
        oled.text("{}: {:4d} {:.2f}V".format(i, readings[i], v), 0, 14 + i * 12)
    oled.show()

def draw_minmax():
    oled.fill(0)
    oled.text("CH MIN  MAX", 0, 0)
    oled.hline(0, 10, 128, 1)
    for i in range(4):
        oled.text("{}: {:4d} {:4d}".format(i, ch_min[i], ch_max[i]), 0, 14 + i * 12)
    oled.show()

oled.fill(0); oled.text("MCP3208 Ready", 8, 28); oled.show(); utime.sleep(1)
print("8-channel ADC logger active.")
print("ch0,ch1,ch2,ch3,ch4,ch5,ch6,ch7")

while True:
    now      = utime.ticks_ms()
    readings = read_all()

    # Update min/max
    for i, v in enumerate(readings):
        ch_min[i] = min(ch_min[i], v)
        ch_max[i] = max(ch_max[i], v)

    # View cycle button
    if btn.value() == 0 and utime.ticks_diff(now, last_btn) > 400:
        view     = (view + 1) % 3
        last_btn = now
        print("View:", VIEW_NAMES[view])

    # Draw active view
    if view == 0:
        draw_bars(readings)
    elif view == 1:
        draw_numeric(readings)
    else:
        draw_minmax()

    # CSV serial output
    print(",".join(str(v) for v in readings))
    utime.sleep_ms(200)
```

## What to Click in MbedO
1. Drag **Pico**, **MCP3208** (or 8 Potentiometers on GP26-GP28 as substitutes), **SSD1306 OLED**, and **Button** onto the canvas.
2. Connect MCP3208 to **GP10-GP13**, OLED to **GP4/GP5**, Button to **GP14**. Connect power and GND.
3. Paste code, click **Run**. Adjust potentiometer sliders; observe the eight-bar chart on OLED. Press Button to switch views.

## Expected Output
```
8-channel ADC logger active.
ch0,ch1,ch2,ch3,ch4,ch5,ch6,ch7
2048,1000,3500,0,4095,2200,800,1500
```

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `cmd = 0b00000110 | ((channel & 0b100) >> 2)` | Builds the first SPI byte for an MCP3208 single-ended channel read using the protocol's start bit and D2 encoding. |
| `((buf[1] & 0x0F) << 8) | buf[2]` | Assembles the 12-bit ADC result from the two-byte response, masking off the null-bit leading the data. |

## Hardware & Safety Concept: SPI ADC vs Built-In ADC
The Pico has only three ADC channels (GP26-GP28), all 12-bit. The MCP3208 adds eight 12-bit channels over a single SPI bus, allowing simultaneous multi-channel data acquisition. Because SPI is synchronous and the MCP3208 holds its conversion result until CS rises, there is no timing ambiguity between channels — each reading is a snapshot.

## Try This! (Challenges)
1. **SD Card CSV**: Write the comma-separated readings to an SD card every 5 seconds using the SPI SD logger from project 155.
2. **Alarm Threshold**: Add user-configurable per-channel thresholds. If any channel exceeds its threshold, flash the OLED header row.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| All channels read 0 or 4095 | CS line not toggling | Verify `cs.value(0)` before and `cs.value(1)` after each transaction. |
| Readings noisy | AGND not connected | Connect BOTH AGND and DGND pins of MCP3208 to the common ground. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [144 - Pico OLED Signal Oscilloscope ADC](144-pico-oled-signal-oscilloscope-adc.md)
- [155 - Pico Environmental Data Station SD Card Logger](155-pico-environmental-data-station-sd-card-logger.md)
