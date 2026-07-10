# 188 - Pico PIO Rotary Encoder

Build a hardware rotary encoder tracker that utilizes the RP2040's Programmable Input/Output (PIO) state machines to sample quadrature signals and detect rotation events in the background.

## Goal
Learn how to implement transition-based edge sampling using a custom PIO program, write conditional jumps based on register comparison (`jmp(x_not_y, ...)`), and decode quadrature encoder states.

## What You Will Build
A PIO-assisted encoder system:
- **Rotary Encoder (GP14 A, GP15 B)**: Quadrature signals monitored by PIO State Machine 0.
- **Active Buzzer (GP16)**: Emits a brief, low-latency click sound on each step.
- **SSD1306 OLED (GP4, GP5)**: Displays the decoded step count, direction of rotation, and the raw PIO transition log.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Rotary Encoder Module | `potentiometer` | Yes (represented by potentiometer) | Yes (EC11 encoder) |
| SSD1306 OLED Display | `oled` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Encoder | Channel A | GP14 | Orange | Quadrature phase A |
| Encoder | Channel B | GP15 | Yellow | Quadrature phase B |
| Encoder | VCC / GND | 3.3V / GND | Red / Black | Power lines |
| Active Buzzer | VCC (+) | GP16 | Blue | Click feedback pin |
| Active Buzzer | GND | GND | Black | Ground return |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The encoder pins GP14 and GP15 must be configured as adjacent inputs for the PIO base pin mapping. The OLED uses GP4/GP5.

## Code
```python
import rp2
from machine import Pin, I2C
import utime, ssd1306

buzzer = Pin(16, Pin.OUT)
buzzer.value(0)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

# --- PIO Program: Quadrature Transition Sampler ---
# Samples the 2-bit state of adjacent pins A and B.
# Compares the new sample (X) with the previous sample (Y).
# If the state changed, it updates Y and pushes the new 2-bit state into the FIFO.
@rp2.asm_pio(in_shiftdir=rp2.PIO.SHIFT_LEFT, autopush=False)
def encoder_sampler():
    wrap_target()
    in_(pins, 2)             # Sample GP14 & GP15 state into ISR
    mov(x, isr)              # Copy ISR contents (new state) to X
    jmp(x_not_y, "changed")  # Compare new state (X) with old state (Y)
    mov(isr, null)           # If unchanged, clear ISR
    jmp("end")
    label("changed")
    mov(y, x)                # Save new state as old state (Y = X)
    push(noblock)            # Push the state change into the RX FIFO
    label("end")
    wrap()

# Pin setup (both input pull-up)
enc_a = Pin(14, Pin.IN, Pin.PULL_UP)
enc_b = Pin(15, Pin.IN, Pin.PULL_UP)

# Instantiate State Machine 0
# in_base specifies GP14 as Pin 0, GP15 becomes Pin 1 automatically
sm = rp2.StateMachine(0, encoder_sampler, freq=1000000, in_base=Pin(14))
sm.active(1)

# Quadrature state transition table (lookup directory)
# Index = (old_state << 2) | new_state
# A CW step is +1, CCW is -1, invalid transitions are 0
TRANSITIONS = [
     0, -1,  1,  0,
     1,  0,  0, -1,
    -1,  0,  0,  1,
     0,  1, -1,  0
]

position = 0
direction = "STOP"
old_state = 0b00

oled.fill(0)
oled.text("PIO Encoder Init", 4, 20)
oled.text("Ready", 4, 36)
oled.show()
utime.sleep(1.0)

print("PIO encoder system active.")

while True:
    # 1. Read transitions from the PIO FIFO buffer
    has_activity = False
    
    # Process all waiting packets in the FIFO
    while sm.rx_fifo() > 0:
        # Get raw 32-bit packet from FIFO. The 2-bit state is in the lower bits.
        fifo_val = sm.get()
        new_state = fifo_val & 0x03
        
        # Calculate transition lookup index
        lookup_idx = (old_state << 2) | new_state
        step = TRANSITIONS[lookup_idx]
        
        if step != 0:
            position += step
            direction = "CW " if step > 0 else "CCW"
            has_activity = True
            
        old_state = new_state
        
    # 2. Click feedback on activity
    if has_activity:
        buzzer.value(1)
        utime.sleep_us(500)
        buzzer.value(0)
        print("Position: {} | Dir: {}".format(position, direction))

    # 3. Update OLED display
    oled.fill(0)
    oled.rect(0, 0, 128, 14, 1)
    oled.text("PIO ENCODER HUD", 4, 3, 0)
    
    oled.text("Position  : {}".format(position), 10, 22, 1)
    oled.text("Direction : {}".format(direction), 10, 36, 1)
    
    fifo_bytes = sm.rx_fifo()
    oled.text("FIFO Queue: {} pkts".format(fifo_bytes), 10, 50, 1)
    
    oled.rect(0, 0, 128, 64, 1)
    oled.show()
    
    utime.sleep_ms(50)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Potentiometer** (encoder simulation), **SSD1306 OLED**, and **Buzzer** onto the canvas.
2. Connect Potentiometer (wiper A, wiper B simulation) to **GP14/GP15**. Connect Buzzer to **GP16** and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the potentiometer controls to simulate transitions. Verify that the position count updates on the OLED and the buzzer clicks.

## Expected Output
Terminal:
```
PIO encoder system active.
Position: 1 | Dir: CW 
Position: 2 | Dir: CW 
Position: 1 | Dir: CCW
```

## Expected Canvas Behavior
* Startup: OLED shows `Position: 0`.
* Simulating steps: The OLED counter increments or decrements depending on the sequence of states.
* Buzzer makes a short, clicking sound.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `in_(pins, 2)` | Reads the logic states of the pin base (GP14) and the next adjacent pin (GP15) into the ISR register. |
| `sm.rx_fifo()` | Returns the number of unread 32-bit words sitting in the State Machine's receive FIFO. |

## Hardware & Safety Concept: Interrupt Pollution vs Hardware FIFO
Reading a high-speed optical rotary encoder (e.g. on a motor shaft spinning at 3000 RPM) using GPIO interrupts can generate up to 20,000 interrupts per second. This creates **interrupt pollution**, consuming 100% of the CPU's processing time and causing the system to hang. Offloading the sampling to a PIO State Machine buffers the raw state changes in a hardware FIFO queue, allowing the CPU to read and decode them in batches, saving CPU overhead.

## Try This! (Challenges)
1. **LED Brightness Control**: Link the encoder position value to adjust a PWM duty cycle on a Red LED on GP13, changing its brightness on clicks.
2. **Speedometer**: Measure the frequency of transition packets in the FIFO and display rotation speed in steps per second.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Position jumps randomly when turning | Quadrature table mismatch | Ensure the transition lookup table values match standard gray code transitions: `00 -> 01` (CW), `00 -> 10` (CCW), etc. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [111 - Rotary Encoder Menu Selector](../intermediate/111-pico-rotary-encoder-menu-selector.md)
- [156 - Pico Interrupt-Driven Encoder Tachometer OLED](../advanced/156-pico-interrupt-driven-encoder-tachometer-oled.md)
