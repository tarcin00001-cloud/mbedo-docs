# 45 - Pico Sound Sensor Serial

Read acoustic noise levels from a sound sensor breakout board and print values to the serial monitor.

## Goal
Learn how to interface analog sound sensors (microphones), read raw voltage peaks, and log sound metrics to the serial terminal in MicroPython.

## What You Will Build
An acoustic noise monitor:
- **Sound Sensor Analog (GP26)**: Measures acoustic volume peaks.
- **Serial Output**: Prints raw analog noise values to the terminal every 200 ms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Sound Sensor Module | `potentiometer` | Yes (represented by potentiometer slider) | Yes (microphone breakout) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Sound Sensor Module | VCC (+) | 3.3V (3V3) | Red | Power line |
| Sound Sensor Module | GND (−) | GND | Black | Ground reference |
| Sound Sensor Module | OUT (Analog) | GP26 | Yellow | Analog signal output |

> **Wiring tip:** Connect the sound sensor's VCC to 3.3V, GND to GND, and the analog output signal pin to GP26.

## Code
```python
from machine import ADC
import utime

sound_sensor = ADC(26) # GP26 = ADC channel 0

print("Sound level monitor online.")

while True:
    raw = sound_sensor.read_u16()
    
    # Calculate relative amplitude (assuming 3.3V supply bias)
    # The microphone board output sits around 1.65V (32768) in silence
    amplitude = raw - 32768
    if amplitude < 0:
        amplitude = -amplitude
        
    print("Raw Reading:", raw, "| Amplitude Offset:", amplitude)
    utime.sleep_ms(200) # Fast polling for noise peaks
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Sound Sensor** (represented by potentiometer) onto the canvas.
2. Connect Sensor VCC to **3.3V**, GND to **GND**, and OUT to **GP26**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the sensor wiper value on the canvas and watch the terminal print amplitude offsets.

## Expected Output
```
Sound level monitor online.
Raw Reading: 32768 | Amplitude Offset: 0
Raw Reading: 45000 | Amplitude Offset: 12232
```

## Expected Canvas Behavior
- The serial terminal prints increasing amplitude offsets as you slide the sensor value away from the center position.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `sound_sensor.read_u16()` | Reads the analog sensor voltage as a 16-bit integer (0–65535). |
| `raw - 32768` | Computes the absolute difference from the silent mid-point bias (1.65V) to calculate volume amplitude. |

## Hardware & Safety Concept: Microphone Bias Volts
Sound waves vibrate a microphone membrane to generate tiny AC voltages. Because analog-to-digital converters (ADCs) on microcontrollers cannot measure negative voltages, microphone breakout boards include an onboard **bias circuit** that offsets the output voltage. This centers the signal in the middle of the ADC range (typically 1.65V or a raw reading of 32768 in silence).

## Try This! (Challenges)
1. **Clap-Activated LED**: Connect an LED on GP15 and configure it to toggle ON and OFF when a loud noise (clap) exceeds a threshold limit of 15000.
2. **Noise Alarm**: Connect a buzzer on GP14 to sound a beep if the noise level remains high for more than 2 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Amplitude is always 0 | Sensor output not biased | Print raw values in silence. If they sit near 0 or 65535, check your wiring and ensure the breakout board is powered correctly. |
| Readings are static and do not change | Signal wire disconnected | Verify the OUT pin is connected to GP26 and not floating. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [11 - Pico Potentiometer Serial](11-pico-potentiometer-serial.md)
- [32 - Pico Brightness Level Indicator](32-pico-brightness-level-indicator.md)
- [41 - Pico Water Level Serial](41-pico-water-level-serial.md)
