# 179 - Pico Multi-Channel ADC Interrupt Readings

Build an analog data acquisition console that uses hardware timer interrupts to read multiple ADC channels at a high frequency, implements software oversampling, and displays filtered results on an LCD.

## Goal
Learn how to use hardware timer interrupts to achieve jitter-free multi-channel analog reads, implement a moving average filter to reduce electrical noise, and update a display from a background buffer in MicroPython.

## What You Will Build
An oversampled three-channel analog monitor:
- **ADC Channel 0 (GP26)**: Reads Potentiometer 1.
- **ADC Channel 1 (GP27)**: Reads Potentiometer 2.
- **ADC Channel 2 (GP28)**: Reads Potentiometer 3.
- **Hardware Timer (Timer 0)**: Fires at 100 Hz (every 10 ms) to trigger the background ADC reads.
- **16x2 I2C LCD (GP4, GP5)**: Displays the moving average of all three channels.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Potentiometers × 3 | `potentiometer` | Yes (three pots) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer 1 | Wiper | GP26 | Yellow | Analog input 0 |
| Potentiometer 2 | Wiper | GP27 | Green | Analog input 1 |
| Potentiometer 3 | Wiper | GP28 | Blue | Analog input 2 |
| All Potentiometers | Left / Right | 3.3V / GND | Red / Black | Voltage reference rails |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the three potentiometer wipers to the Pico's three ADC pins: GP26, GP27, and GP28. The I2C LCD uses GP4/GP5.

## Code
```python
from machine import Pin, ADC, I2C, Timer
import utime
from machine_lcd import I2cLcd

# Analog inputs
adc0 = ADC(26)
adc1 = ADC(27)
adc2 = ADC(28)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Buffer size for moving average
BUFFER_SIZE = 8

# Ring buffers for each channel
buf0 = [0] * BUFFER_SIZE
buf1 = [0] * BUFFER_SIZE
buf2 = [0] * BUFFER_SIZE
buf_idx = 0

sample_count = 0
adc_timer = Timer()

def adc_timer_callback(timer):
    """Interrupt service routine (ISR) fired at 100 Hz (every 10 ms)."""
    global buf_idx, sample_count
    
    # Read raw 16-bit analog inputs
    buf0[buf_idx] = adc0.read_u16()
    buf1[buf_idx] = adc1.read_u16()
    buf2[buf_idx] = adc2.read_u16()
    
    # Increment ring buffer index
    buf_idx = (buf_idx + 1) % BUFFER_SIZE
    sample_count += 1

# Initialize LCD
lcd.clear()
lcd.putstr("ADC Interrupt\nSampler Ready...")
utime.sleep(1.0)

# Initialize timer at 100 Hz (periodic)
adc_timer.init(freq=100, mode=Timer.PERIODIC, callback=adc_timer_callback)
print("100 Hz ADC timer interrupt active.")

while True:
    # 1. Calculate moving average from buffers
    # Doing math in the main thread keeps the interrupt handler fast!
    avg0 = sum(buf0) // BUFFER_SIZE
    avg1 = sum(buf1) // BUFFER_SIZE
    avg2 = sum(buf2) // BUFFER_SIZE
    
    # 2. Display average values (scaled to 0-99 range for layout fit)
    val0 = avg0 * 100 // 65536
    val1 = avg1 * 100 // 65536
    val2 = avg2 * 100 // 65536
    
    lcd.clear()
    lcd.putstr("C0:{} C1:{} C2:{}\n".format(val0, val1, val2))
    lcd.putstr("Samples: {}".format(sample_count))
    
    print("CH0:{} | CH1:{} | CH2:{} | Total Samples: {}".format(avg0, avg1, avg2, sample_count))
    utime.sleep_ms(200)  # Update display 5 times per second
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **three Potentiometers**, and **I2C LCD** onto the canvas.
2. Connect Potentiometers to **GP26**, **GP27**, and **GP28**. Connect LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the three potentiometer sliders. Watch the values on the LCD update smoothly. The sample count increments at 100 Hz.

## Expected Output
Terminal:
```
100 Hz ADC timer interrupt active.
CH0:32768 | CH1:16384 | CH2:49152 | Total Samples: 100
CH0:32800 | CH1:16400 | CH2:49100 | Total Samples: 120
```

## Expected Canvas Behavior
* The LCD displays the three scaled channel values: e.g. `C0:50 C1:25 C2:75`.
* The `Samples` counter increments rapidly in the background.
* Adjusting the sliders changes the displayed values.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `adc0.read_u16()` | Acquires a raw 16-bit analog sample (0 to 65535) from GP26. |
| `sum(buf0) // BUFFER_SIZE` | Computes the average of the last 8 samples to filter out transient noise. |

## Hardware & Safety Concept: High-Speed Sampling and ISR Limits
When sampling analog signals, any jitter (variation in sample timing) introduces noise into the data. Running the ADC reads inside a hardware timer interrupt guarantees precise, regular sampling intervals. However, because display writes and string formatting are slow, they must never run inside the interrupt handler — the ISR must only read the hardware and write to a RAM buffer, leaving the calculations to the main thread.

## Try This! (Challenges)
1. **Critical Overlimit Alarm**: Flash a warning LED on GP15 if any of the three analog channels exceeds an average value of 55000 (83% scale).
2. **Frequency Doubler**: Modify the timer frequency to 200 Hz, and check if the buffer size can be increased to 16 for better filtering.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Values bounce rapidly on LCD | Buffer size too small | Increase `BUFFER_SIZE` to 16 or 32 to damp out rapid slider changes. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [31 - Potentiometer ADC Read](../../beginner/31-potentiometer-adc-read-gp26-adc0.md)
- [158 - Pico MCP3208 Eight-Channel ADC Logger OLED](../advanced/158-pico-mcp3208-eight-channel-adc-logger-oled.md)
