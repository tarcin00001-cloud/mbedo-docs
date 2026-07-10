# 165 - Pico Industrial Scale Sorter

Build an automated sorting conveyor that weighs packages on a load cell and routes them using separate sorting gates based on weight classifications.

## Goal
Learn how to interface HX711 load cell amplifiers, display sorting metrics and categories on OLED screens, and actuate dual-relay sorting gates in MicroPython.

## What You Will Build
A multi-bin industrial weight sorter:
- **HX711 & Load Cell (DOUT GP14, SCK GP15)**: Measures package weight.
- **Relay 1 (GP10)**: Reject gate for underweight packages (< 100g).
- **Relay 2 (GP11)**: Reject gate for overweight packages (> 200g).
- **SSD1306 OLED (GP4, GP5)**: Displays live weight and active bin routing (e.g. "UNDERWEIGHT", "STANDARD", or "OVERWEIGHT").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HX711 Amplifier Module | `hx711` | Yes | Yes |
| Load Cell Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| Relay Modules | `relay` | Yes (two relays) | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HX711 | DOUT | GP14 | Orange | Data output line |
| HX711 | PD_SCK | GP15 | Yellow | Serial clock line |
| HX711 | VCC / GND | 3.3V / GND | Red / Black | Module power |
| Relay 1 | IN | GP10 | Green | Underweight reject gate control |
| Relay 2 | IN | GP11 | Blue | Overweight reject gate control |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect HX711 DOUT to GP14 and SCK to GP15. Relay control inputs connect to GP10 and GP11. The OLED shares the I2C Bus 0 on GP4/GP5. Ensure all ground connections are shared.

## Code
```python
from machine import Pin, I2C
import utime, ssd1306

# HX711 simple bit-banged driver class
class HX711:
    def __init__(self, dout_pin, sck_pin):
        self.dout = Pin(dout_pin, Pin.IN)
        self.sck = Pin(sck_pin, Pin.OUT)
        self.sck.value(0)
        self.offset = 0
        self.scale_factor = 420.0

    def raw_read(self):
        # Wait for DOUT to go low (indicates conversion complete)
        for _ in range(500):
            if self.dout.value() == 0:
                break
            utime.sleep_us(100)
        else:
            return 0
            
        val = 0
        for _ in range(24):
            self.sck.value(1)
            utime.sleep_us(1)
            val = (val << 1) | self.dout.value()
            self.sck.value(0)
            utime.sleep_us(1)
            
        # 25th pulse for 128 gain (Channel A)
        self.sck.value(1)
        utime.sleep_us(1)
        self.sck.value(0)
        utime.sleep_us(1)
        
        # Sign-extend 24-bit to 32-bit signed
        if val & 0x800000:
            val -= 0x1000000
        return val

    def tare(self):
        total = 0
        for _ in range(10):
            total += self.raw_read()
            utime.sleep_ms(10)
        self.offset = total / 10

    def get_units(self, count=5):
        total = 0
        for _ in range(count):
            total += self.raw_read()
            utime.sleep_ms(10)
        avg = total / count
        return (avg - self.offset) / self.scale_factor

# Sorter pins
relay1 = Pin(10, Pin.OUT)
relay2 = Pin(11, Pin.OUT)
relay1.value(0)
relay2.value(0)

# I2C OLED
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

# Initialize scale
scale = HX711(14, 15)
oled.fill(0)
oled.text("Zeroing scale...", 10, 28)
oled.show()
scale.tare()

# Target weight boundaries (grams)
UNDER_LIMIT = 100.0
OVER_LIMIT  = 200.0

print("Industrial weighing sorter online.")

while True:
    weight = scale.get_units(5)
    if weight < 0:
        weight = 0.0

    oled.fill(0)
    
    # Draw header banner
    oled.fill_rect(0, 0, 128, 14, 1)
    oled.text("SORTING CONVEYOR", 4, 3, 0)
    
    oled.text("Weight: {:.1f} g".format(weight), 10, 20, 1)

    if weight > 10.0:  # Check only if an item is on the scale
        if weight < UNDER_LIMIT:
            # Underweight sorting action
            relay1.value(1)
            relay2.value(0)
            oled.text("CLASS : UNDER", 10, 34, 1)
            oled.text("Gate 1: ACTIVE", 10, 48, 1)
        elif weight > OVER_LIMIT:
            # Overweight sorting action
            relay1.value(0)
            relay2.value(1)
            oled.text("CLASS : OVER", 10, 34, 1)
            oled.text("Gate 2: ACTIVE", 10, 48, 1)
        else:
            # Standard weight - pass through
            relay1.value(0)
            relay2.value(0)
            oled.text("CLASS : STANDARD", 10, 34, 1)
            oled.text("Gates : PASS", 10, 48, 1)
    else:
        # Empty
        relay1.value(0)
        relay2.value(0)
        oled.text("CLASS : EMPTY", 10, 34, 1)

    oled.rect(0, 0, 128, 64, 1)
    oled.show()
    
    print("Weight: {:.1f}g".format(weight))
    utime.sleep_ms(500)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HX711**, **Load Cell** (represented by potentiometer), **two Relays**, and **SSD1306 OLED** onto the canvas.
2. Connect HX711 to **GP14/GP15**, Relays to **GP10/GP11**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the weight potentiometer to simulate package weights and check if the correct reject gate activates.

## Expected Output
Terminal:
```
Industrial weighing sorter online.
Weight: 0.0g
Weight: 85.3g
Weight: 145.0g
Weight: 250.0g
```

## Expected Canvas Behavior
* Empty Scale (< 10g): OLED reads `CLASS: EMPTY`. Both relays are OFF.
* Standard Package (150g): OLED reads `CLASS: STANDARD` / `Gates: PASS`. Both relays are OFF.
* Underweight Package (50g): OLED reads `CLASS: UNDER` / `Gate 1: ACTIVE`. Relay 1 turns ON.
* Overweight Package (250g): OLED reads `CLASS: OVER` / `Gate 2: ACTIVE`. Relay 2 turns ON.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `scale.tare()` | Zeroes out the weight measurement, subtracting the dead weight of the empty scale platform. |
| `weight < UNDER_LIMIT` | Evaluates if the package is underweight to route it to reject bin 1. |

## Hardware & Safety Concept: Industrial Sorting Gates
Industrial sorting systems route packages to different bins (such as sorting by size, weight, or ZIP code). Diverter gates open quickly to guide items into their correct channels. Emergency limit monitoring is critical: if a gate jams, safety sensors shut down the conveyor line to prevent package blockages.

## Try This! (Challenges)
1. **Auditory Alert**: Connect a buzzer on GP12 and sound a beep when a package is sorted to either reject bin.
2. **Interactive Calibration**: Add a push button on GP16 that zeroes the scale when pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Both gates activate at the same time | Code logic error | Ensure the check conditions use `if / elif` structures so only one relay can activate at a time. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [138 - Pico Stepper Motor CNC Positioner LCD](138-pico-stepper-motor-cnc-positioner-lcd.md)
- [158 - Pico MCP3208 Eight-Channel ADC Logger OLED](158-pico-mcp3208-eight-channel-adc-logger-oled.md)
