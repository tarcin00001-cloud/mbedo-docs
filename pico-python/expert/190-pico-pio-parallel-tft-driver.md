# 190 - Pico PIO Parallel Bus Driver

Build a high-speed 8-bit parallel bus transmitter that utilizes a custom PIO program to write a byte of data across eight GPIO pins simultaneously in a single clock cycle.

## Goal
Learn how to configure multi-pin output maps in PIO (`out(pins, 8)`), manipulate consecutive GPIO registers, and toggle latch clocks using side-set parameters in MicroPython.

## What You Will Build
An 8-bit parallel bus interface:
- **8 Parallel Data Pins (GP0 to GP7)**: Outputs 8-bit data values simultaneously.
- **Write Clock Pin (GP8)**: Acts as the write clock latch (`WR`), toggled automatically by PIO side-set.
- **Potentiometer (GP26)**: Generates 8-bit values (0 to 255) to transmit over the bus.
- **SSD1306 OLED (GP4, GP5)**: Displays the active binary byte pattern and transmit statistics.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| SSD1306 OLED Display | `oled` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| LEDs × 8 (or LED Bar Graph) | `led` | Yes (to visualize GP0-GP7 states) | Yes (for bus visualization) |
| 330 Ω Resistors × 8 | `resistor` | Optional | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Data Bus D0–D7 | Anodes (via 330 Ω) | GP0 to GP7 | Green wires | 8-bit parallel data lines |
| Data Bus Cathodes | Ground | GND | Black | Shared return rail |
| Latch Clock (WR) | Signal | GP8 | Orange | Latch clock output line |
| Potentiometer | Wiper | GP26 | Yellow | Input value generator |
| Potentiometer | Left / Right | 3.3V / GND | Red / Black | Voltage reference rails |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** GP0 through GP7 must be wired consecutively to represent bits 0 to 7 of the parallel bus. The write latch clock is GP8. The OLED uses GP4/GP5.

## Code
```python
import rp2
from machine import Pin, ADC, I2C
import utime, ssd1306

pot = ADC(26)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

# --- PIO Program: 8-Bit Parallel Bus Transmitter ---
# 1. Shifts 8 bits of data from the FIFO directly to 8 output pins.
# 2. Automatically pulls the WR latch clock (side-set) LOW.
# 3. Pulls the WR clock HIGH to latch the data into the receiver.
@rp2.asm_pio(
    out_init=(rp2.PIO.OUT_LOW,) * 8, 
    sideset_init=rp2.PIO.OUT_HIGH,
    out_shiftdir=rp2.PIO.SHIFT_RIGHT,
    autopull=True,
    pull_thresh=8
)
def parallel_bus():
    wrap_target()
    out(pins, 8)             .side(0) [1]  # Drive 8 data pins, pull WR clock LOW, wait 1 cycle
    nop()                    .side(1) [1]  # Keep data stable, pull WR clock HIGH (latch), wait 1 cycle
    wrap()

# Configure data pins GP0-GP7 as outputs
data_pins = [Pin(i, Pin.OUT) for i in range(8)]
# Configure clock pin GP8 as output
wr_pin = Pin(8, Pin.OUT)

# Instantiate State Machine 0
# out_base=Pin(0) binds the 8 out pins to GP0-GP7
# sideset_base=Pin(8) binds the .side() call to GP8
sm = rp2.StateMachine(0, parallel_bus, freq=10000000, out_base=Pin(0), sideset_base=Pin(8))
sm.active(1)

def write_bus(val):
    """Pushes a single byte onto the parallel bus."""
    # Write the byte to the State Machine TX FIFO.
    # The autopull feature automatically pulls it when out() executes.
    sm.put(val & 0xFF)

oled.fill(0)
oled.text("Parallel Bus", 10, 20)
oled.text("Ready", 10, 36)
oled.show()
utime.sleep(1.0)

print("PIO parallel bus driver active.")
write_count = 0

while True:
    raw_val = pot.read_u16()
    
    # Scale pot (0-65535) to 8-bit byte range (0-255)
    byte_val = raw_val * 255 // 65536
    
    # Write to parallel bus via PIO
    write_bus(byte_val)
    write_count += 1
    
    # Convert byte value to binary string for visualization
    bin_str = ""
    for i in range(7, -1, -1):
        bin_str += "1" if (byte_val & (1 << i)) else "0"
        
    # Update OLED HUD
    oled.fill(0)
    oled.rect(0, 0, 128, 14, 1)
    oled.text("PIO PARALLEL BUS", 4, 3, 0)
    
    oled.text("Value Dec: {}".format(byte_val), 10, 20, 1)
    oled.text("Value Bin: {}".format(bin_str), 10, 34, 1)
    oled.text("Writes   : {}".format(write_count), 10, 48, 1)
    
    oled.rect(0, 0, 128, 64, 1)
    oled.show()
    
    utime.sleep_ms(100)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **SSD1306 OLED**, **Potentiometer**, and **eight LEDs** (or bar graph) onto the canvas.
2. Connect LEDs to **GP0 through GP7** in order. Connect clock line (LED) to **GP8**. Connect Potentiometer to **GP26**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the Potentiometer. Observe the 8 LEDs change to reflect the binary representation of the pot value. The GP8 LED will flash rapidly (simulated clock).

## Expected Output
Terminal:
```
PIO parallel bus driver active.
```
(On OLED: displays values like `Value Dec: 128` / `Value Bin: 10000000`.)

## Expected Canvas Behavior
* Sliding Potentiometer to maximum (255): All 8 LEDs (GP0-GP7) turn ON.
* Sliding to minimum (0): All 8 LEDs turn OFF.
* Sliding to middle (128): GP7 turns ON, GP0-GP6 turn OFF.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `out(pins, 8)` | Writes 8 bits of data from the shift register onto the 8 consecutive pins starting at `out_base` (GP0-GP7). |
| `sideset_base=Pin(8)` | Maps the assembly `.side(val)` operations to GP8, driving the latch clock. |

## Hardware & Safety Concept: Serial vs Parallel Communication
Serial buses (like SPI or I2C) transfer data one bit at a time over a single wire, requiring multiple clock cycles per byte. Parallel buses transfer an entire byte (or 16/32 bits) simultaneously across multiple wires in a single clock cycle. This makes parallel buses extremely fast, but they are highly sensitive to **clock skew** (propagation delays where bits arrive at different times). Hardware registers and latch clocks ensure all bits are stable before the receiver reads them.

## Try This! (Challenges)
1. **Counter Sweep**: Write a Python loop that automatically counts from 0 to 255 and back, generating a visual counting pattern on the LEDs.
2. **Frequency Doubler**: Increase the State Machine clock frequency to 20 MHz and observe if the bus remains stable.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Data bits are swapped or out of order | Pin mapping incorrect | Verify that GP0 is wired to D0, GP1 to D1, up to GP7 to D7. Parallel buses must use consecutive hardware pins. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [04 - Multiple LED Chase](../../beginner/04-multiple-led-chase.md)
- [187 - Pico PIO NeoPixel Driver](187-pico-pio-neopixel-driver.md)
