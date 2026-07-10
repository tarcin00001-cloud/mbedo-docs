# 06 - Pico Active Buzzer

Drive an active buzzer to produce a repeating beep pattern — your first audio output project.

## Goal
Learn how to toggle a GPIO digital output pin to power an active buzzer, and use `utime.sleep()` to create timed beep patterns in MicroPython.

## What You Will Build
A repeating buzzer beep:
- **Active Buzzer (GP15)**: Beeps ON for 200 ms, then OFF for 800 ms, repeating continuously.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Active Buzzer (5V or 3.3V) | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Active Buzzer | VCC (+) / Signal | GP15 | Red | GPIO drives the buzzer directly when HIGH |
| Active Buzzer | GND (−) | GND | Black | Buzzer return path to ground |

> **Wiring tip:** An **active buzzer** contains its own internal oscillator — it emits a fixed tone whenever its VCC pin is driven HIGH. No PWM or frequency control is needed. Simply toggle the pin ON/OFF to start/stop the beep.
>
> **Active vs Passive:** If your buzzer does NOT make sound when connected directly to 3.3 V, it is a **passive buzzer** — see [07 - Pico Passive Buzzer Tone](07-pico-tone.md) instead.
>
> **5V buzzers on a 3.3 V Pico:** Most active buzzers rated for 5V will still produce sound when driven from the Pico's 3.3 V GPIO pin, though the tone may be slightly quieter. For full volume on hardware, power the buzzer from VBUS (5V) and use a transistor (e.g. 2N2222) or MOSFET switch driven by the GPIO pin.

## Code
```python
from machine import Pin
import utime

buzzer = Pin(15, Pin.OUT)

while True:
    buzzer.value(1)       # Buzzer ON (beep)
    utime.sleep(0.2)      # Beep for 200 ms
    buzzer.value(0)       # Buzzer OFF
    utime.sleep(0.8)      # Silence for 800 ms
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and an **Active Buzzer** onto the canvas.
2. Connect the Buzzer VCC (+) pin to **GP15** and GND (−) to **GND**.
3. Paste the code, select **MicroPython** mode, and click **Run**.

## Expected Output
A short beep sounds every second (200 ms on, 800 ms off), repeating continuously.

## Expected Canvas Behavior
- The buzzer component on the canvas activates (shows active state) for 200 ms, then deactivates for 800 ms.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Pin(15, Pin.OUT)` | Configures GP15 as a digital output to drive the buzzer. |
| `buzzer.value(1)` | Drives GP15 HIGH (3.3 V), activating the buzzer's internal oscillator. |
| `utime.sleep(0.2)` | Holds the buzzer ON for 200 milliseconds. |
| `buzzer.value(0)` | Drives GP15 LOW (0 V), silencing the buzzer. |
| `utime.sleep(0.8)` | Holds the buzzer OFF for 800 milliseconds (pause between beeps). |

## Hardware & Safety Concept: Active vs Passive Buzzers
| Feature | Active Buzzer | Passive Buzzer |
| --- | --- | --- |
| Internal oscillator | Yes — fixed tone | No — requires external signal |
| Control method | Simple ON/OFF (digital) | PWM frequency required |
| Typical use | Alarms, notifications | Melodies, variable tones |
| Identification | Usually marked with a "+" on top | No polarity markings |

## Try This! (Challenges)
1. **SOS Pattern**: Encode the Morse code SOS pattern (3 short, 3 long, 3 short beeps).
2. **Button Doorbell**: Wire a button to GP14 and only beep while the button is pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| No sound | Passive buzzer used | Use the PWM approach in [07 - Pico Passive Buzzer Tone](07-pico-tone.md) instead. |
| Buzzer beeps continuously | GND and VCC swapped | Check polarity — the VCC (+) leg connects to GP15, not GND. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [07 - Pico Passive Buzzer Tone](07-pico-tone.md)
- [08 - Pico Button Buzzer](08-pico-button-buzzer.md)
- [15 - Pico SOS Buzzer](15-pico-sos-buzzer.md)
