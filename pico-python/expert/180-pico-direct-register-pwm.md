# 180 - Pico Direct Register PWM Configuration

Build a hardware register control dashboard that bypasses MicroPython's high-level classes, directly manipulating the RP2040 memory-mapped registers using `machine.mem32` to control an LED's PWM brightness.

## Goal
Learn how to perform direct register access (register-level programming) in MicroPython, configure the RP2040 IO multiplexer (GPIO_CTRL) and PWM peripheral registers directly, and understand memory-mapped I/O.

## What You Will Build
A direct-register LED driver:
- **Red LED (GP15)**: Brightness is controlled by direct register manipulation of PWM Slice 7.
- **Potentiometer (GP26)**: Reads analog input using standard ADC to dynamically set the compare register.
- **SSD1306 OLED (GP4, GP5)**: Displays the raw register values (CSR, TOP, CC, CTRL) in hex.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| SSD1306 OLED Display | `oled` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| Red LED | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Red LED | Anode (via 330 Ω) | GP15 | Red | PWM hardware output (Slice 7 Channel B) |
| Red LED | Cathode | GND | Black | Ground return |
| Potentiometer | Wiper | GP26 | Yellow | Analog control input |
| Potentiometer | Left / Right | 3.3V / GND | Red / Black | Reference voltage rails |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** GP15 belongs to the RP2040's hardware PWM Slice 7, Channel B. The OLED screen uses shared I2C pins GP4/GP5.

## Code
```python
from machine import Pin, ADC, I2C, mem32
import utime, ssd1306

pot = ADC(26)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

# --- Register Base Addresses ---
IO_BANK0_BASE = 0x40014000
PWM_BASE      = 0x40050000

# GP15 Mux control register address:
# GPIO15_CTRL = IO_BANK0_BASE + 15 * 8 + 4 = 0x4001407C
GPIO15_CTRL   = IO_BANK0_BASE + 0x7C

# GP15 is on PWM Slice 7, Channel B.
# Each PWM slice register block is 24 bytes (0x18) wide.
# Slice 7 offset = 7 * 24 = 168 (0xA8).
SLICE7_BASE = PWM_BASE + 0xA8

PWM_CH7_CSR = SLICE7_BASE + 0x00   # Control & Status
PWM_CH7_DIV = SLICE7_BASE + 0x04   # Clock Divider
PWM_CH7_CTR = SLICE7_BASE + 0x08   # Counter value
PWM_CH7_CC  = SLICE7_BASE + 0x0C   # Compare (Duty cycle)
PWM_CH7_TOP = SLICE7_BASE + 0x10   # Wrap (Frequency)

# --- Initialize Registers Directly ---

# 1. Configure GP15 pin function select to ALT_PWM (Function 4)
# Write 4 to the FUNCSEL bits (bits 4:0) of GPIO15_CTRL
mem32[GPIO15_CTRL] = 4

# 2. Configure PWM Slice 7 Wrap (Frequency)
# A wrap value of 10000 at 125 MHz clock gives a 12.5 kHz frequency
mem32[PWM_CH7_TOP] = 10000

# 3. Configure PWM Slice 7 Clock Divider
# Write 16 (bits 11:4 integer, 3:0 fraction) to DIV register (1.0 scale)
mem32[PWM_CH7_DIV] = 16 << 4

# 4. Initialize Compare Register (Duty cycle) to 0
# CC register layout: bits 31:16 for Channel B, 15:0 for Channel A
# GP15 is Channel B, so we write to the upper 16 bits.
mem32[PWM_CH7_CC] = 0 << 16

# 5. Enable PWM Slice 7 (CSR bit 0 = 1)
mem32[PWM_CH7_CSR] = 1

print("Direct register PWM configurations active on GP15.")

while True:
    raw_pot = pot.read_u16()
    
    # Scale pot (0-65535) to match compare range (0-10000)
    duty_limit = raw_pot * 10000 // 65536
    
    # Write directly to compare register Upper 16 bits (Channel B)
    # Clear and set UPPER 16 bits, keeping lower 16 bits (Channel A) intact
    current_cc = mem32[PWM_CH7_CC] & 0x0000FFFF
    mem32[PWM_CH7_CC] = current_cc | (duty_limit << 16)
    
    # Read registers back to display
    csr_val = mem32[PWM_CH7_CSR]
    top_val = mem32[PWM_CH7_TOP]
    cc_val  = mem32[PWM_CH7_CC]
    ctrl_val = mem32[GPIO15_CTRL]
    
    # Draw Register HUD on OLED
    oled.fill(0)
    oled.rect(0, 0, 128, 14, 1)
    oled.text("REGISTER REGISTER", 4, 3, 0)
    
    oled.text("GP15CTRL: 0x{:08X}".format(ctrl_val), 2, 20, 1)
    oled.text("PWM_CSR : 0x{:08X}".format(csr_val), 2, 32, 1)
    oled.text("PWM_TOP : 0x{:08X}".format(top_val), 2, 44, 1)
    oled.text("PWM_CC  : 0x{:08X}".format(cc_val), 2, 56, 1)
    
    oled.show()
    utime.sleep_ms(100)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Potentiometer**, **Red LED**, and **SSD1306 OLED** onto the canvas.
2. Connect Potentiometer to **GP26**, LED to **GP15**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the Potentiometer control. Observe the Red LED brightness change, and verify that the upper bits of `PWM_CC` in the OLED hex display change in response.

## Expected Output
Terminal:
```
Direct register PWM configurations active on GP15.
```
(On OLED: displays hex values like `PWM_CC: 0x13880000` showing upper channel B compare active.)

## Expected Canvas Behavior
* Boot: Red LED is OFF. OLED displays registers.
* Slide Potentiometer up: Red LED glows progressively brighter.
* The hex value of `PWM_CC` on the screen changes (e.g. `0x00000000` -> `0x27100000` at full brightness).

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `mem32[GPIO15_CTRL] = 4` | Configures the multiplexer select bits for GP15 to map to function index 4 (PWM). |
| `mem32[PWM_CH7_CC] = ...` | Writes directly to the hardware compare register memory address, altering duty cycle. |

## Hardware & Safety Concept: Memory-Mapped I/O and Peripheral Registers
Microcontrollers do not have special instructions to access hardware peripherals (like timers, PWM, or UART). Instead, they map these hardware controls to specific memory addresses called **registers**. Writing to or reading from these addresses changes the hardware configuration directly. Using `machine.mem32` allows developers to bypass standard drivers to implement custom features or maximize performance.

## Try This! (Challenges)
1. **Clock Divider Control**: Add a second potentiometer on GP27 and map its value to write directly to `PWM_CH7_DIV` (frequency scaling).
2. **Frequency Sweeper**: Write a loop that increments `PWM_CH7_TOP` directly from 1000 to 20000 back and forth, creating a frequency chirp.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Pico hangs or crashes on run | Illegal address written | double-check register base addresses. Writing to unmapped memory locations triggers a HardFault error, crashing the CPU. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [02 - LED Blink Rate Sweep](../../beginner/02-led-blink-rate-sweep-changing-sleep-values.md)
- [157 - Pico PWM Fan Controller Tachometer LCD](../advanced/157-pico-pwm-fan-controller-tachometer-lcd.md)
