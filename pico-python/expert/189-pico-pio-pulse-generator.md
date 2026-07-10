# 189 - Pico PIO Pulse Generator

Build an adjustable high-frequency pulse generator that uses a custom PIO program to generate non-blocking square waves on a GPIO pin, controlled by a potentiometer.

## Goal
Learn how to write a register-controlled dual-loop PIO program in MicroPython, load parameters from the TX FIFO buffer asynchronously, and generate precise pulse-width signals.

## What You Will Build
An adjustable clock pulse generator:
- **Signal Output (GP15)**: Generates square wave pulses driven by PIO State Machine 0.
- **Potentiometer (GP26)**: Adjusts the pulse frequency dynamically from 10 Hz to 50 kHz.
- **Duty Cycle Button (GP13)**: Cycles the duty cycle between 25%, 50%, and 75%.
- **SSD1306 OLED (GP4, GP5)**: Displays the calculated frequency, duty cycle, and raw loop cycle counts.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| SSD1306 OLED Display | `oled` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| LED (optional indicator) | `led` | Yes | Yes (low-frequency visual check) |
| 330 Ω Resistor | `resistor` | Optional | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Signal Output | GP15 (via LED) | GP15 | Yellow | High-frequency pulse output |
| Button (Duty) | Terminal 1 | GP13 | White | Duty cycle selector button |
| Button (Duty) | Terminal 2 | GND | Black | Ground return |
| Potentiometer | Wiper | GP26 | Green | Frequency control analog input |
| Potentiometer | Left / Right | 3.3V / GND | Red / Black | Voltage reference rails |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect GP15 to an LED or oscilloscope channel. The duty cycle button is on GP13. The OLED uses GP4/GP5.

## Code
```python
import rp2
from machine import Pin, ADC, I2C
import utime, ssd1306

pot = ADC(26)
btn_duty = Pin(13, Pin.IN, Pin.PULL_UP)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

# --- PIO Program: Dual-loop Pulse Generator ---
# Pulls the HIGH cycle count, drives sideset pin HIGH, and decrements X.
# Pulls the LOW cycle count, drives sideset pin LOW, and decrements X.
# If FIFO is empty, pull(noblock) keeps the previous value in the OSR.
@rp2.asm_pio(sideset_init=rp2.PIO.OUT_LOW)
def pulse_generator():
    wrap_target()
    pull(noblock)             # Get HIGH duration cycles from FIFO (remains in OSR if empty)
    mov(x, osr)               # Load into X register
    label("high_loop")
    nop()             .side(1) # Keep signal output HIGH
    jmp(x_dec, "high_loop")   # Loop until X is 0 (2 cycles per loop)
    
    pull(noblock)             # Get LOW duration cycles from FIFO
    mov(x, osr)               # Load into X
    label("low_loop")
    nop()             .side(0) # Keep signal output LOW
    jmp(x_dec, "low_loop")    # Loop until X is 0 (2 cycles per loop)
    wrap()

# Pin configuration
pulse_out = Pin(15, Pin.OUT)

# Instantiate State Machine 0
# Set base clock frequency of State Machine to 1 MHz (1 cycle = 1 us)
# sideset_base maps the .side() call in assembly to GP15
sm = rp2.StateMachine(0, pulse_generator, freq=1000000, sideset_base=Pin(15))
sm.active(1)

# Default starting parameters (1000 cycles total = 1000 us = 1 kHz frequency)
high_cycles = 500
low_cycles = 500
duty_mode = 1  # 0=25%, 1=50%, 2=75%
duty_options = [25, 50, 75]

def update_pulse_parameters(total_period_us, duty_pct):
    """Calculates loop cycles and pushes them to the PIO FIFO."""
    global high_cycles, low_cycles
    
    # Calculate durations in microseconds (since SM clock is 1 MHz)
    # Each loop instruction is 2 cycles (2 us)
    high_time_us = total_period_us * duty_pct // 100
    low_time_us = total_period_us - high_time_us
    
    # Cycles to load in registers (loops decrement by 1)
    high_cycles = max(1, (high_time_us // 2) - 1)
    low_cycles = max(1, (low_time_us // 2) - 1)
    
    # Push parameters to TX FIFO
    # PIO expects HIGH count, then LOW count
    sm.put(high_cycles)
    sm.put(low_cycles)

# Initial update: 1000 us period (1 kHz) at 50% duty
update_pulse_parameters(1000, 50)

oled.fill(0)
oled.text("PIO Pulse Gen", 4, 20)
oled.text("Online", 4, 36)
oled.show()
utime.sleep(1.0)

print("PIO pulse generator active.")

last_btn = 0

while True:
    now = utime.ticks_ms()
    raw_pot = pot.read_u16()
    
    # Map potentiometer to period from 10000 us (100 Hz) to 20 us (50 kHz)
    target_period_us = 20 + (raw_pot * 9980 // 65536)
    target_freq_hz = 1000000 // target_period_us
    
    # Handle Duty Cycle Button (GP13)
    if btn_duty.value() == 0 and utime.ticks_diff(now, last_btn) > 400:
        duty_mode = (duty_mode + 1) % 3
        last_btn = now
        print("Duty Cycle set to:", duty_options[duty_mode], "%")
        
    # Update PIO State Machine parameters
    update_pulse_parameters(target_period_us, duty_options[duty_mode])
    
    # Update Display HUD
    oled.fill(0)
    oled.rect(0, 0, 128, 14, 1)
    oled.text("PIO PULSE HUD", 10, 3, 0)
    
    oled.text("Freq : {} Hz".format(target_freq_hz), 10, 20, 1)
    oled.text("Duty : {} %".format(duty_options[duty_mode]), 10, 32, 1)
    oled.text("H_Reg: {} ({}us)".format(high_cycles, high_cycles*2), 10, 44, 1)
    oled.text("L_Reg: {} ({}us)".format(low_cycles, low_cycles*2), 10, 56, 1)
    
    oled.rect(0, 0, 128, 64, 1)
    oled.show()
    
    utime.sleep_ms(100)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Potentiometer**, **Push Button**, **LED**, and **SSD1306 OLED** onto the canvas.
2. Connect Potentiometer to **GP26**, Button to **GP13**, LED to **GP15**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the potentiometer to change the output frequency. Click the button to switch between 25%, 50%, and 75% duty cycles. Observe the LED brightness and OLED display statistics change.

## Expected Output
Terminal:
```
PIO pulse generator active.
Duty Cycle set to: 75 %
Duty Cycle set to: 25 %
```

## Expected Canvas Behavior
* LED (GP15) flashes at low frequencies (slide potentiometer to maximum resistance/minimum frequency).
* Adjusting the frequency slider changes the speed values on the OLED.
* Pressing the button changes the LED brightness ratio (duty cycle check).

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `pull(noblock)` | Attempts to fetch a data value from the TX FIFO buffer. If the buffer is empty, it does not wait, leaving the contents of the OSR register unchanged. |
| `jmp(x_dec, "high_loop")` | Decrements the X register by 1. If X is non-zero, jumps back to `high_loop`. |

## Hardware & Safety Concept: Jitter-Free Clock Generators
Traditional software PWM relies on CPU clock cycles and interrupt execution times. If the CPU becomes busy writing to an LCD screen or handling serial UART traffic, the software-generated pulse widths will stretch and jitter. Bypassing the CPU and using the PIO hardware state machine guarantees jitter-free clocks, which is critical for driving sensitive synchronous components (like CCD sensors, stepping motors, or SPI ADCs).

## Try This! (Challenges)
1. **Siren Sweep Sweep**: Write a loop in Python that continuously shifts the target period up and down, generating a sound chirp if a speaker is connected to GP15.
2. **Frequency Presets**: Configure Button B (GP14) to cycle through fixed frequency presets: 1 kHz, 5 kHz, 10 kHz, and 20 kHz.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Output stays static HIGH or LOW | FIFO starved or logic loop stuck | Ensure that `update_pulse_parameters` writes both high and low cycle counts to the State Machine using two `sm.put()` calls. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [02 - LED Blink Rate Sweep](../../beginner/02-led-blink-rate-sweep-changing-sleep-values.md)
- [175 - Pico Hardware Timer PWM Tone Sweeps](175-pico-hardware-timer-pwm-sweeps.md)
