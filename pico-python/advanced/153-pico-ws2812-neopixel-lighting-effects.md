# 153 - Pico WS2812 NeoPixel Lighting Effects

Build a NeoPixel LED strip controller that cycles through six animated lighting effects (rainbow, chase, breathing, color wipe, sparkle, and fire) selectable via a push button, with speed control via a potentiometer.

## Goal
Learn how to drive WS2812B addressable LEDs using the Pico's PIO state machine, implement multiple animation patterns as generator-based effects, and control playback speed with an analog potentiometer in MicroPython.

## What You Will Build
A NeoPixel effects controller:
- **WS2812B LED Strip (GP0)**: 16-pixel addressable RGB strip.
- **Potentiometer (GP26)**: Controls animation speed.
- **Button (GP13)**: Cycles through six lighting effects.
- **I2C 16x2 LCD (GP4, GP5)**: Displays the active effect name and speed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| WS2812B NeoPixel Strip (16 LEDs) | `neopixel` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 300–500 Ω Resistor | `resistor` | Optional in MbedO | Yes (data line protection) |
| 1000 µF Capacitor | — | — | Yes (strip power decoupling) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| WS2812B Strip | DIN (data in) | GP0 | Orange | NeoPixel data (via 300 Ω resistor) |
| WS2812B Strip | VCC (+5V) | 5V (VBUS) | Red | Strip power — dedicate supply for > 8 LEDs |
| WS2812B Strip | GND | GND | Black | Shared ground (connect at strip AND Pico) |
| Potentiometer | Wiper | GP26 | Yellow | Speed control ADC |
| Button | Terminal 1 / 2 | GP13 / GND | White / Black | Effect cycle button |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Place a 300–500 Ω resistor in series with the WS2812B DIN line to prevent ringing. For strips longer than 8 LEDs, power the strip from a separate 5V supply, connecting only ground back to the Pico. All grounds are shared.

## Code
```python
from machine import Pin, ADC, I2C
import utime, neopixel, random
from machine_lcd import I2cLcd

NUM_LEDS = 16
np  = neopixel.NeoPixel(Pin(0), NUM_LEDS)
pot = ADC(26)
btn = Pin(13, Pin.IN, Pin.PULL_UP)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

EFFECT_NAMES = ["RAINBOW ", "CHASE   ", "BREATHE ", "WIPE    ", "SPARKLE ", "FIRE    "]
effect   = 0
last_btn = 0

def hsv_to_rgb(h, s, v):
    """Convert HSV (0-255 each) to RGB tuple."""
    if s == 0: return (v, v, v)
    i = h // 43
    f = (h - i * 43) * 6
    p = v * (255 - s) // 255
    q = v * (255 - s * f // 255) // 255
    t = v * (255 - s * (255 - f) // 255) // 255
    combos = [(v,t,p),(q,v,p),(p,v,t),(p,q,v),(t,p,v),(v,p,q)]
    return combos[i % 6]

def rainbow(offset, brightness=40):
    for i in range(NUM_LEDS):
        hue = (i * 255 // NUM_LEDS + offset) % 256
        np[i] = hsv_to_rgb(hue, 255, brightness)
    np.write()

def chase(offset, color=(0, 60, 120)):
    for i in range(NUM_LEDS):
        np[i] = color if i == offset % NUM_LEDS else (0, 0, 0)
    np.write()

def breathe(phase, color=(0, 40, 80)):
    import math
    v = int((math.sin(phase / 50.0) + 1) * 60)
    for i in range(NUM_LEDS):
        np[i] = (v * color[0] // 255, v * color[1] // 255, v * color[2] // 255)
    np.write()

def color_wipe(pos, color=(80, 0, 40)):
    if pos < NUM_LEDS:
        np[pos] = color
    np.write()

def sparkle(brightness=60):
    for i in range(NUM_LEDS):
        np[i] = (0, 0, 0)
    idx = random.randint(0, NUM_LEDS - 1)
    np[idx] = (brightness, brightness, brightness)
    np.write()

FIRE_PALETTE = [(0,0,0),(80,0,0),(120,20,0),(160,60,0),(200,120,0),(255,200,0),(255,255,150)]
def fire():
    for i in range(NUM_LEDS):
        heat = random.randint(0, 6)
        np[i] = FIRE_PALETTE[heat]
    np.write()

phase   = 0
wipe_pos = 0

lcd.clear(); lcd.putstr("NeoPixel FX"); utime.sleep(1)
print("NeoPixel effects controller active.")

while True:
    now  = utime.ticks_ms()
    spd  = pot.read_u16()
    delay_ms = max(10, 200 - spd * 200 // 65535)

    # Button: next effect
    if btn.value() == 0 and utime.ticks_diff(now, last_btn) > 400:
        effect   = (effect + 1) % len(EFFECT_NAMES)
        last_btn = now
        phase    = 0; wipe_pos = 0
        print("Effect:", EFFECT_NAMES[effect])

    # Draw current effect
    if effect == 0:
        rainbow(phase)
    elif effect == 1:
        chase(phase)
    elif effect == 2:
        breathe(phase)
    elif effect == 3:
        color_wipe(wipe_pos)
        wipe_pos = (wipe_pos + 1) % (NUM_LEDS * 2)
    elif effect == 4:
        sparkle()
    elif effect == 5:
        fire()

    phase = (phase + 1) % 256

    # LCD
    lcd.clear()
    lcd.putstr(EFFECT_NAMES[effect])
    lcd.move_to(0, 1)
    lcd.putstr("Speed: {:3d} ms".format(delay_ms))

    utime.sleep_ms(delay_ms)
```

## What to Click in MbedO
1. Drag **Pico**, **NeoPixel Strip**, **Potentiometer**, **Push Button**, and **I2C LCD** onto the canvas.
2. Connect NeoPixel DIN to **GP0**, Potentiometer to **GP26**, Button to **GP13**, LCD to **GP4/GP5**. Connect power and GND.
3. Paste code, click **Run**. The RAINBOW effect starts. Press Button to cycle to CHASE, BREATHE, etc. Slide pot to change speed.

## Expected Output
```
NeoPixel effects controller active.
Effect: CHASE
Effect: BREATHE
```
(On LCD: current effect name and delay in ms. On strip: animated lighting pattern.)

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `hsv_to_rgb(hue, 255, brightness)` | Converts HSV colour space to RGB, allowing smooth hue cycling for rainbow effects. |
| `delay_ms = max(10, 200 - spd * 200 // 65535)` | Maps potentiometer 0–65535 to delay 200ms–10ms: turning up = faster animation. |

## Hardware & Safety Concept: WS2812B Power Requirements
Each WS2812B LED can draw up to 60mA at full white (R+G+B all at 255). A 16-LED strip at full white requires up to 960mA — beyond what a USB port safely supplies. The code uses reduced brightness values (40–80 instead of 255) to keep current within safe limits. For full-brightness applications, use a dedicated 5V 2A supply.

## Try This! (Challenges)
1. **BT Remote Control**: Accept `R`, `C`, `B`, `W`, `S`, `F` commands from an HC-05 to select effects remotely.
2. **Beat Detection**: Sample the microphone ADC and trigger a sparkle burst when the RMS level spikes above a threshold.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Strip flickers or shows wrong colours | Missing data resistor | Add 300-500 Ω in series with the DIN data line. |
| Only first few LEDs work | Power supply too weak | Power the strip from a separate 5V 1A+ supply, connecting ground to Pico. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [75 - Pico LED Strip WS2812](../intermediate/75-pico-led-strip-ws2812.md)
- [116 - Pico RGB LED Color Mixer](../intermediate/116-pico-rgb-led-color-mixer.md)
