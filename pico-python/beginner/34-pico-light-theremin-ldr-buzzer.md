# 34 - Pico Light Theremin LDR Buzzer

Build a light-controlled theremin that adjusts the pitch (frequency) of a passive buzzer dynamically based on ambient light levels.

## Goal
Learn how to map an analog light sensor (LDR) voltage range to a customized frequency range (in Hertz) to control the tone pitch of a passive buzzer in MicroPython.

## What You Will Build
A light-controlled pitch theremin:
- **LDR Sensor (GP26)**: Measures ambient light intensity.
- **Passive Buzzer (GP15)**: Adjusts pitch frequency from 200 Hz to 3000 Hz based on the light falling on the LDR.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LDR Photoresistor | `ldr` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes (passive required) |
| 10 kΩ Resistor | `resistor` | Optional in MbedO | Yes (voltage divider) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR | Pin 1 | 3.3V (3V3) | Red | Power line |
| LDR | Pin 2 | GP26 | Yellow | Analog input signal |
| 10 kΩ Resistor | Either leg | GP26 to GND | Black | Pull-down resistor |
| Passive Buzzer | Signal (+) | GP15 | Red | PWM frequency output pin |
| Passive Buzzer | GND (−) | GND | Black | Shared return path to ground |

> **Wiring tip:** The LDR and 10 kΩ resistor form a voltage divider connected to GP26. The passive buzzer requires a pulsing signal to vibrate its internal membrane. Connect its (+) terminal to GP15 and GND terminal to GND.

## Code
```python
from machine import Pin, ADC, PWM
import utime

ldr = ADC(26)          # GP26 = ADC channel 0
buzzer = PWM(Pin(15))  # GP15 = PWM output
buzzer.duty_u16(32768) # 50% duty cycle (constant volume)

while True:
    raw = ldr.read_u16()
    
    # Map raw value (0 - 65535) to frequency range (200 Hz - 3000 Hz)
    freq = 200 + int(raw * 2800 / 65535)
    
    # Apply frequency
    buzzer.freq(freq)
    
    print("Light Level:", raw, "| Freq:", freq, "Hz")
    utime.sleep_ms(50) # Polling update delay
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **LDR**, and **Passive Buzzer** onto the canvas.
2. Connect LDR Pin 1 to **3.3V** and Pin 2 to **GP26**.
3. Connect Buzzer Signal (+) to **GP15** and GND (−) to **GND**.
4. Paste the code, select **MicroPython** mode, and click **Run**.
5. Adjust the LDR slider on the canvas and observe the buzzer pitch frequency changing.

## Expected Output
```
Light Level: 12000 | Freq: 712 Hz
Light Level: 45000 | Freq: 2121 Hz
```

## Expected Canvas Behavior
- The passive buzzer component vibrates or indicates active sound at varying frequency rates depending on the LDR slider position.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `buzzer.duty_u16(32768)` | Sets the duty cycle to exactly 50%. This generates a clean square wave at the target frequency, creating maximum tone volume. |
| `200 + int(raw * 2800 / 65535)` | Linearly scales the 16-bit input range (0–65535) to a frequency output range of 200 Hz to 3000 Hz. |
| `buzzer.freq(freq)` | Dynamically changes the active PWM output frequency to adjust the pitch. |

## Hardware & Safety Concept: Optical Theremins
Optical theremins are classic electronic instruments that use light sensors to control audio frequency generators. They allow users to play music without physical contact, simply by waving their hands over the light sensors to shade them.

## Try This! (Challenges)
1. **Dynamic Volume**: Add a second LDR on GP27 to adjust the buzzer's volume (PWM duty cycle) based on light levels.
2. **Reverse Pitch**: Adjust the code so that shading the LDR increases the pitch frequency instead of lowering it.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer click/pops without tone | Frequency set too low | Verify the mapping offset is at least 100 Hz. |
| Pitch does not change | Buzzer is active type | Active buzzers have fixed oscillators and ignore input frequencies. Ensure you use a passive buzzer. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [07 - Pico Passive Buzzer Tone](07-pico-tone.md)
- [10 - Pico LDR Light Sensor Serial](10-pico-ldr-serial.md)
- [18 - Pico Potentiometer Buzzer Frequency](18-pico-potentiometer-buzzer-frequency.md)
