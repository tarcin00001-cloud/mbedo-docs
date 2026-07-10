# 187 - Pico PIO NeoPixel Driver

Build an addressable LED strip controller that uses the RP2040's hardware Programmable Input/Output (PIO) state machines to generate precise 800 kHz WS2812B data timings in the background.

## Goal
Learn how to write assembly-like PIO state machine instructions using MicroPython's `@rp2.asm_pio` decorator, configure peripheral side-set pins, and drive addressable LEDs.

## What You Will Build
A PIO-driven WS2812B controller:
- **WS2812B LED Strip (GP0)**: An addressable RGB strip controlled by PIO State Machine 0.
- **Potentiometer (GP26)**: Adjusts the color hue dynamically.
- **SSD1306 OLED (GP4, GP5)**: Displays active PIO configurations and the raw GRB binary output buffers.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| WS2812B NeoPixel Strip (16 LEDs) | `neopixel` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| 300–500 Ω Resistor | `resistor` | Optional | Yes (data line protection) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| WS2812B Strip | DIN (data in) | GP0 | Orange | NeoPixel data (via 300 Ω resistor) |
| WS2812B Strip | VCC | 5V (VBUS) | Red | Power lines |
| WS2812B Strip | GND | GND | Black | Ground reference |
| Potentiometer | Wiper | GP26 | Yellow | Hue adjustment analog input |
| Potentiometer | Left / Right | 3.3V / GND | Red / Black | Reference voltage rails |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** WS2812B data requires very fast transition times. Place a 300 Ω resistor in series with GP0. Connect the OLED screen to GP4/GP5.

## Code
```python
import rp2
from machine import Pin, ADC, I2C
import utime, ssd1306

pot = ADC(26)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

NUM_LEDS = 16

# --- PIO State Machine definition for 800kHz WS2812 protocol ---
# Standard WS2812 bits: 1.25us total duration.
# 8 MHz state machine clock frequency = 125ns per clock cycle.
# 10 cycles total per bit.
# Bit '1': 6 cycles high, 4 cycles low
# Bit '0': 3 cycles high, 7 cycles low
@rp2.asm_pio(sideset_init=rp2.PIO.OUT_LOW, out_shiftdir=rp2.PIO.SHIFT_LEFT, autopull=True, pull_thresh=24)
def ws2812_led():
    wrap_target()
    label("bit_loop")
    out(x, 1)               .side(0) [2]  # Pull 1 bit, keep low for 3 cycles (1 + 2 delay)
    jmp(not_x, "do_zero")   .side(1) [2]  # If bit is 0, jump. If 1, drive high for 3 cycles
    jmp("bit_loop")         .side(1) [3]  # If 1, keep high for 4 more cycles, wrap loop
    label("do_zero")
    nop()                   .side(0) [3]  # If 0, drive low for 4 cycles (1 + 3 delay)
    wrap()

# Instantiate the state machine on State Machine 0
# sideset_base maps GP0 to the .side() instructions in assembly
sm = rp2.StateMachine(0, ws2812_led, freq=8000000, sideset_base=Pin(0))
sm.active(1)

def hsv_to_rgb(h, s, v):
    """Convert HSV to GRB format for WS2812B."""
    if s == 0: return (v, v, v)
    i = h // 43
    f = (h - i * 43) * 6
    p = v * (255 - s) // 255
    q = v * (255 - s * f // 255) // 255
    t = v * (255 - s * (255 - f) // 255) // 255
    
    combos = [(v,t,p),(q,v,p),(p,v,t),(p,q,v),(t,p,v),(v,p,q)]
    r, g, b = combos[i % 6]
    # WS2812B expects Green, Red, Blue color order
    return g, r, b

def show_pixels(pixels):
    """Pushes the pixel list into the PIO State Machine FIFO buffer."""
    buf = bytearray(len(pixels) * 3)
    idx = 0
    for g, r, b in pixels:
        buf[idx] = g
        buf[idx+1] = r
        buf[idx+2] = b
        idx += 3
    # Write the 24-bit color integers directly to the SM TX FIFO
    sm.put(buf)

# Main LED strip array
led_data = [(0, 0, 0)] * NUM_LEDS

oled.fill(0)
oled.text("PIO LED Driver", 10, 20)
oled.text("Initialising...", 10, 36)
oled.show()
utime.sleep(1.0)

print("PIO NeoPixel driver active.")

while True:
    raw_val = pot.read_u16()
    
    # Map pot (0-65535) to hue scale (0-255)
    hue = raw_val * 255 // 65536
    
    # Calculate colors: base color + slight offset per LED for gradient
    for i in range(NUM_LEDS):
        h = (hue + i * 8) % 256
        led_data[i] = hsv_to_rgb(h, 255, 30)  # Brightness kept low for safety
        
    # Send colors to PIO
    show_pixels(led_data)
    
    # Display state
    oled.fill(0)
    oled.rect(0, 0, 128, 14, 1)
    oled.text("PIO CONFIG HUD", 10, 3, 0)
    
    oled.text("SM Frequency: 8MHz", 2, 20, 1)
    oled.text("Active Pin  : GP0", 2, 32, 1)
    oled.text("Hue Offset  : {}".format(hue), 2, 44, 1)
    oled.text("TX FIFO     : ACTIVE", 2, 56, 1)
    
    oled.rect(0, 0, 128, 64, 1)
    oled.show()
    
    utime.sleep_ms(50)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **NeoPixel Strip**, **Potentiometer**, and **SSD1306 OLED** onto the canvas.
2. Connect NeoPixel Data to **GP0**, Potentiometer to **GP26**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the Potentiometer control. Verify that the LED strip displays a shifting rainbow gradient and the OLED updates its config display.

## Expected Output
Terminal:
```
PIO NeoPixel driver active.
```

## Expected Canvas Behavior
* Startup: OLED shows `PIO CONFIG HUD`.
* The 16-LED strip lights up with a smooth, moving color gradient.
* Adjusting the potentiometer wiper changes the base hue of the color spectrum on the strip.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `@rp2.asm_pio(...)` | Compiles Python code containing PIO assembly instructions into binary state machine code. |
| `sm.put(buf)` | Pushes the raw byte array into the state machine's transmit buffer (TX FIFO). |

## Hardware & Safety Concept: RP2040 PIO (Programmable I/O)
WS2812B LEDs require a single-wire control signal with microsecond-level precision. Standard CPU software loops cannot guarantee this level of timing, especially if interrupts occur. The RP2040 contains two **PIO blocks**, each containing four independent **State Machines**. These state machines run assembly code in the background, generating exact timings without using any CPU execution cycles.

## Try This! (Challenges)
1. **Breathing Strobe**: Modify the code to scale the HSV Value (brightness) up and down using a sine wave in the background.
2. **Double Pin Side-Set**: Modify the `@rp2.asm_pio` sideset configuration to toggle a second pin (e.g. GP1) in sync with the data line to create an oscilloscope trigger.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The LED strip displays garbled colors or flickers | Timing mismatch | The state machine frequency must be exactly 8000000 (8 MHz) to scale the assembly cycles to match the WS2812B specifications. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [75 - Pico LED Strip WS2812](../intermediate/75-pico-led-strip-ws2812.md)
- [153 - Pico WS2812 NeoPixel Lighting Effects](../advanced/153-pico-ws2812-neopixel-lighting-effects.md)
