# 18 - Pico Potentiometer Buzzer Frequency

Adjust the pitch (frequency) of a passive buzzer dynamically using a potentiometer knob.

## Goal
Learn how to map an analog input voltage range to a customized frequency range (in Hertz) to control the tone pitch of a passive buzzer in MicroPython.

## What You Will Build
A variable pitch synth:
- **Potentiometer (GP26)**: Reads knob adjustments.
- **Passive Buzzer (GP15)**: Adjusts pitch frequency from 100 Hz to 2000 Hz based on the potentiometer wiper position.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes (passive required) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Left Pin (Pin 1) | GND | Black | Ground reference |
| Potentiometer | Wiper (Pin 2 - centre) | GP26 | Yellow | Analog input voltage |
| Potentiometer | Right Pin (Pin 3) | 3.3V (3V3) | Red | High voltage reference |
| Passive Buzzer | Signal (+) | GP15 | Red | PWM frequency output pin |
| Passive Buzzer | GND (−) | GND | Black | Shared return path to ground |

> **Wiring tip:** The passive buzzer requires a pulsing signal to vibrate its internal membrane. Connect its (+) terminal to GP15 and GND terminal to GND. The potentiometer middle wiper pin goes to GP26.

## Code
```python
from machine import Pin, ADC, PWM
import utime

pot = ADC(26)          # GP26 = ADC channel 0
buzzer = PWM(Pin(15))  # GP15 = PWM output
buzzer.duty_u16(32768) # 50% duty cycle (constant volume)

while True:
    raw = pot.read_u16()
    
    # Map raw value (0 - 65535) to frequency range (100 Hz - 2000 Hz)
    freq = 100 + int(raw * 1900 / 65535)
    
    # Apply frequency
    buzzer.freq(freq)
    
    utime.sleep_ms(50) # Polling update delay
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Potentiometer**, and **Passive Buzzer** onto the canvas.
2. Connect Potentiometer Pin 1 to **GND**, Pin 2 to **GP26**, and Pin 3 to **3.3V**.
3. Connect Buzzer Signal (+) to **GP15** and GND (−) to **GND**.
4. Paste the code, select **MicroPython** mode, and click **Run**.
5. Adjust the potentiometer slider on the canvas and observe the buzzer pitch frequency changing.

## Expected Output
The buzzer pitch rises smoothly to a high squeal as you slide the potentiometer up, and drops to a low hum when you slide it down.

## Expected Canvas Behavior
- The passive buzzer component vibrates or indicates active sound at varying frequency rates depending on the potentiometer wiper position.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `buzzer.duty_u16(32768)` | Sets the duty cycle to exactly 50%. This generates a clean square wave at the target frequency, creating maximum tone volume. |
| `100 + int(raw * 1900 / 65535)` | Linearly scales the 16-bit input range (0–65535) to a frequency output range of 100 Hz to 2000 Hz. |
| `buzzer.freq(freq)` | Dynamically changes the active PWM output frequency to adjust the pitch. |

## Hardware & Safety Concept: Audible Frequency Bounds
The human hearing range is roughly **20 Hz to 20,000 Hz**. Microcontrollers can easily generate these frequencies. Piezoelectric buzzers perform best in the middle range (typically **500 Hz to 4,000 Hz**). Generating frequencies below 100 Hz may result in quiet clicks rather than tones due to mechanical limits.

## Try This! (Challenges)
1. **Theremin Mode**: Add an LDR light sensor on GP27 and mix both sensor values to create a dual-axis light/rotary theremin.
2. **Step Notes**: Restrict the output frequency to match only standard Western music notes instead of continuous slides.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer click/pops without tone | Frequency set too low | Verify the mapping offset is at least 100 Hz. |
| Pitch does not change | Buzzer is active type | Active buzzers have fixed oscillators and ignore input frequencies. Ensure you use a passive buzzer. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [07 - Pico Passive Buzzer Tone](07-pico-tone.md)
- [11 - Pico Potentiometer Serial](11-pico-potentiometer-serial.md)
