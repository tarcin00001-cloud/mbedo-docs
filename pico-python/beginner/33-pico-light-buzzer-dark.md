# 33 - Pico Light Buzzer Dark

Build a darkness alarm that sounds a warning buzzer when the room becomes completely dark.

## Goal
Learn how to read an analog light sensor (LDR) and implement threshold logic to trigger an active buzzer alarm in MicroPython.

## What You Will Build
A darkness alert system:
- **LDR Sensor (GP26)**: Monitors ambient light levels.
- **Active Buzzer (GP15)**: Automatically sounds a warning beep if the light level drops below a set threshold, indicating total darkness.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LDR Photoresistor | `ldr` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | Optional in MbedO | Yes (voltage divider) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR | Pin 1 | 3.3V (3V3) | Red | Power line |
| LDR | Pin 2 | GP26 | Yellow | Analog input signal |
| 10 kΩ Resistor | Either leg | GP26 to GND | Black | Pull-down resistor |
| Active Buzzer | VCC (+) | GP15 | Red | Alarm output pin |
| Active Buzzer | GND (−) | GND | Black | Shared return path to ground |

> **Wiring tip:** The LDR and 10 kΩ resistor form a voltage divider connected to GP26. Connect the active buzzer to GP15 and GND.

## Code
```python
from machine import Pin, ADC
import utime

ldr = ADC(26) # GP26 = ADC channel 0
buzzer = Pin(15, Pin.OUT)

# Threshold: values below this indicate total darkness (raw ADC range 0 - 65535)
DARK_LIMIT = 8000

buzzer.value(0) # Ensure buzzer starts silent

while True:
    val = ldr.read_u16()
    print("Light Level:", val)

    if val < DARK_LIMIT:
        print("ALERT: Total darkness detected!")
        # Pulse warning buzzer
        buzzer.value(1)
        utime.sleep_ms(300)
        buzzer.value(0)
        utime.sleep_ms(300)
    else:
        buzzer.value(0)
        utime.sleep_ms(200) # Normal polling speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **LDR**, and **Active Buzzer** onto the canvas.
2. Connect LDR Pin 1 to **3.3V** and Pin 2 to **GP26**.
3. Connect Buzzer VCC (+) to **GP15** and GND (−) to **GND**.
4. Paste the code, select **MicroPython** mode, and click **Run**.
5. Move the LDR slider down (dark) to see the buzzer activate.

## Expected Output
```
Light Level: 45000
Light Level: 5000
ALERT: Total darkness detected!
```

## Expected Canvas Behavior
- The buzzer component on the canvas pulses active when the LDR slider is moved down past the threshold.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `val < DARK_LIMIT` | Checks if the measured light level is below the threshold, indicating dark conditions. |
| `buzzer.value(1)` | Sounds the alarm buzzer when darkness is detected. |

## Hardware & Safety Concept: Darkness Alerts
Darkness alerts are commonly used in safety applications like lighting control, solar tracking, and security systems. Using simple threshold logic allows the system to trigger actions automatically when light conditions drop below safety limits.

## Try This! (Challenges)
1. **Light Warning**: Invert the logic to sound the buzzer when a bright light is detected in a dark room (for drawer/closet alarms).
2. **Flash and Ring Alert**: Connect an LED on GP13 to flash in sync with the buzzer alarm pulses.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer sounds continuously | Threshold set too high | Print LDR values to the terminal to find the ambient light level, then adjust the `DARK_LIMIT` in your code. |
| Buzzer does not sound | LDR wired incorrectly | Verify that the LDR and fixed resistor are configured as a voltage divider at GP26. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [06 - Pico Active Buzzer](06-pico-buzzer.md)
- [10 - Pico LDR Light Sensor Serial](10-pico-ldr-serial.md)
- [22 - Pico Night Light LDR](22-pico-night-light-ldr.md)
