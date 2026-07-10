# 145 - Pico Rotary Encoder Menu System OLED

Build an interactive OLED menu system driven by a rotary encoder that scrolls through menu items, selects parameters, adjusts values with rotation, and confirms changes with the encoder push button.

## Goal
Learn how to decode a quadrature rotary encoder using interrupts, implement a hierarchical OLED menu with list navigation and value editing modes, and commit edited values on button press in MicroPython.

## What You Will Build
An OLED menu navigation system:
- **Rotary Encoder (GP14 CLK, GP15 DT, GP13 SW)**: Rotates to scroll list or adjust values; pushes to select or confirm.
- **SSD1306 OLED (GP4, GP5)**: Renders a 3-visible-item menu list with a cursor highlight.
- Four editable parameters: LED Brightness, Fan Speed, Alarm Threshold, Backlight.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Rotary Encoder Module (KY-040) | `encoder` | Yes | Yes |
| SSD1306 OLED Display (128×64) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Rotary Encoder | CLK (A) | GP14 | Orange | Quadrature channel A |
| Rotary Encoder | DT (B) | GP15 | Yellow | Quadrature channel B |
| Rotary Encoder | SW (Push) | GP13 | Blue | Push button (active LOW) |
| Rotary Encoder | VCC | 3.3V (3V3) | Red | Module power |
| Rotary Encoder | GND | GND | Black | Ground reference |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| SSD1306 OLED | VCC / GND | 3.3V / GND | Red / Black | OLED power |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect CLK to GP14, DT to GP15, and SW to GP13. The encoder module usually includes pull-up resistors. If using a bare encoder, add 10 kΩ pull-ups on CLK and DT. Connect OLED to GP4/GP5.

## Code
```python
from machine import Pin, I2C
import utime, ssd1306

i2c  = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

clk_pin = Pin(14, Pin.IN, Pin.PULL_UP)
dt_pin  = Pin(15, Pin.IN, Pin.PULL_UP)
sw_pin  = Pin(13, Pin.IN, Pin.PULL_UP)

# Menu structure: (label, min, max, current_value, step)
MENU = [
    ["LED Bright", 0, 100, 75, 5],
    ["Fan Speed",  0, 100, 50, 10],
    ["Alarm Thr",  20, 80, 35, 1],
    ["Backlight",  0, 100, 100, 5],
]

MODE_NAVIGATE = 0
MODE_EDIT     = 1

mode     = MODE_NAVIGATE
cursor   = 0       # Highlighted menu item index
top_item = 0       # First visible item (scroll offset)
VISIBLE  = 4       # Items visible on screen

# Encoder state
last_clk   = clk_pin.value()
last_btn   = 0
enc_delta  = 0

def read_encoder():
    """Non-interrupt polling encoder. Returns +1, -1, or 0."""
    global last_clk, enc_delta
    clk = clk_pin.value()
    if clk != last_clk:
        if dt_pin.value() != clk:
            enc_delta = 1   # CW
        else:
            enc_delta = -1  # CCW
        last_clk = clk
        return enc_delta
    return 0

def draw_menu():
    oled.fill(0)
    title = "< Edit >" if mode == MODE_EDIT else "  MENU  "
    oled.text(title, 20, 0)
    oled.hline(0, 10, 128, 1)

    for i in range(VISIBLE):
        idx = top_item + i
        if idx >= len(MENU):
            break
        y = 14 + i * 13
        label, mn, mx, val, _ = MENU[idx]
        line  = "{}: {:3d}".format(label[:8], val)
        if idx == cursor:
            oled.fill_rect(0, y - 1, 128, 12, 1)  # Highlight bar
            oled.text(line, 2, y, 0)              # Inverted text
        else:
            oled.text(line, 2, y, 1)
    oled.show()

draw_menu()
print("Rotary encoder menu active. Rotate to scroll, push to select/confirm.")

while True:
    now = utime.ticks_ms()
    delta = read_encoder()

    if delta != 0:
        if mode == MODE_NAVIGATE:
            cursor = max(0, min(len(MENU) - 1, cursor + delta))
            # Scroll if cursor moves out of view
            if cursor < top_item:
                top_item = cursor
            elif cursor >= top_item + VISIBLE:
                top_item = cursor - VISIBLE + 1
        elif mode == MODE_EDIT:
            item = MENU[cursor]
            item[3] = max(item[1], min(item[2], item[3] + delta * item[4]))
        draw_menu()

    # Button press: toggle navigate/edit
    if sw_pin.value() == 0 and utime.ticks_diff(now, last_btn) > 300:
        last_btn = now
        if mode == MODE_NAVIGATE:
            mode = MODE_EDIT
            print("Editing:", MENU[cursor][0])
        else:
            mode = MODE_NAVIGATE
            print("Saved {} = {}".format(MENU[cursor][0], MENU[cursor][3]))
        draw_menu()
        utime.sleep_ms(200)

    utime.sleep_ms(5)
```

## What to Click in MbedO
1. Drag **Pico**, **Rotary Encoder**, and **SSD1306 OLED** onto the canvas.
2. Connect CLK to **GP14**, DT to **GP15**, SW to **GP13**, and OLED to **GP4/GP5**. Connect power and GND.
3. Paste code, click **Run**. Simulate encoder rotation with the CLK/DT sliders; simulate button with SW.
4. Rotate to move cursor, push to enter Edit mode, rotate to adjust value, push to save.

## Expected Output
```
Rotary encoder menu active. Rotate to scroll, push to select/confirm.
Editing: LED Bright
Saved LED Bright = 80
Editing: Fan Speed
Saved Fan Speed = 60
```
(On OLED: a 4-item list with the highlighted item rendered as an inverted (white-on-black) bar.)

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `oled.fill_rect(0, y-1, 128, 12, 1)` + `oled.text(..., 0)` | Draws a white highlight bar and renders text in inverted colour (black-on-white) for the selected item. |
| `top_item = cursor - VISIBLE + 1` | Scroll-follows cursor: keeps the selected item visible by adjusting the top of the visible window. |

## Hardware & Safety Concept: Quadrature Encoder Decoding
A rotary encoder produces two square waves (A and B channels) 90° out of phase. The direction of rotation determines which channel leads. If A leads B, rotation is clockwise; if B leads A, it is counter-clockwise. Reading only channel A transitions and comparing to B state at that instant gives direction with minimal code.

## Try This! (Challenges)
1. **Long Press**: Detect a long press (> 800 ms) of the encoder button to reset the selected parameter to its default value.
2. **Apply Values**: Connect a real LED on GP16 and update its PWM duty when "LED Bright" is saved.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Cursor jumps two items per click | Encoder outputs two pulses per detent | Divide `delta` by 2, or only act on rising edge of CLK. |
| Button registers multiple presses | Mechanical bounce | Increase debounce delay from 300 ms to 500 ms. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [144 - Pico OLED Signal Oscilloscope ADC](144-pico-oled-signal-oscilloscope-adc.md)
- [140 - Pico Full Weather Console DHT22 BMP180 LDR OLED](140-pico-full-weather-console-dht22-bmp180-ldr-oled.md)
