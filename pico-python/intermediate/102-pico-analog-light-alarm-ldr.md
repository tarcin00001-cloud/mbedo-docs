# 102 - Pico Analog Light Alarm LDR

Build a light level security threshold alarm that flashes a warning LED and sounds a pulsing warning buzzer when ambient light levels fall below comfort limits.

## Goal
Learn how to read analog inputs from an LDR light sensor, implement threshold alarm logic, and control warning LEDs and buzzers in MicroPython.

## What You Will Build
A light level threshold alarm:
- **LDR Sensor (GP26)**: Monitors ambient light levels.
- **LED (GP14)**: Flashes during alarm states.
- **Active Buzzer (GP15)**: Sounds a pulsing warning siren when triggered.
- **Reset Button (GP13)**: Silences the alarm once the light returns to safe limits.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LDR Photoresistor | `ldr` | Yes | Yes |
| LED (any colour) | `led` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Tactile Push Button | `button` | Yes (reset switch) | Yes |
| 10 kΩ Resistor | `resistor` | Optional in MbedO | Yes (voltage divider) |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR | Pin 1 | 3.3V (3V3) | Red | Power line |
| LDR | Pin 2 | GP26 | Yellow | Analog input signal |
| 10 kΩ Resistor | Either leg | GP26 to GND | Black | Pull-down resistor |
| Reset Button | Terminal 1 | GP13 | White | Reset input (reads LOW on press) |
| Reset Button | Terminal 2 | GND | Black | Shorts GP13 to GND |
| LED | Anode (+, longer leg) | GP14 | Orange | Warning indicator output pin |
| LED | Cathode (−, shorter leg) | GND | Black | Shared return path to ground |
| 330 Ω Resistor | Either leg | In series with LED Anode | — | Limits LED current |
| Active Buzzer | VCC (+) | GP15 | Red | Alarm sound output pin |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The LDR and 10 kΩ resistor form a voltage divider connected to GP26. Connect the reset button to GP13, LED anode to GP14, and active buzzer VCC to GP15. All grounds are shared.

## Code
```python
from machine import Pin, ADC
import utime

ldr       = ADC(26) # GP26 = ADC channel 0
btn_reset = Pin(13, Pin.IN, Pin.PULL_UP)
warning_l = Pin(14, Pin.OUT)
buzzer    = Pin(15, Pin.OUT)

# Threshold: alarm triggers if light level falls below this (raw scale 0 - 65535)
DARK_LIMIT = 15000

alarm_state = False
warning_l.value(0)
buzzer.value(0)

print("Light alarm active.")

while True:
    raw = ldr.read_u16()
    reset_pressed = (btn_reset.value() == 0)
    
    # Check for trigger (Set)
    if raw < DARK_LIMIT and not alarm_state:
        alarm_state = True
        print(">> WARNING: Light level too low! Alarm active. Reading:", raw)
        
    # Check for disarm (Reset)
    if reset_pressed and alarm_state:
        alarm_state = False
        warning_l.value(0)
        buzzer.value(0)
        print(">> Light alarm reset.")
        utime.sleep(1) # Lockout delay
        
    # Pulsing warning siren
    if alarm_state:
        warning_l.value(1)
        buzzer.value(1)
        utime.sleep_ms(200)
        warning_l.value(0)
        buzzer.value(0)
        utime.sleep_ms(200)
    else:
        utime.sleep_ms(200) # Normal polling speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **LDR**, **Push Button** (reset switch), **LED**, and **Active Buzzer** onto the canvas.
2. Connect LDR Pin 1 to **3.3V** and Pin 2 to **GP26**.
3. Connect Reset Button to **GP13**, LED to **GP14**, and Buzzer to **GP15**. Connect all grounds.
4. Paste the code, select **MicroPython** mode, and click **Run**.
5. Slide the LDR value down past the threshold to trigger the alarm. Click Reset to disarm.

## Expected Output
```
Light alarm active.
>> WARNING: Light level too low! Alarm active. Reading: 12000
>> Light alarm reset.
```

## Expected Canvas Behavior
- The LED and buzzer components on the canvas pulse ON and OFF rapidly when the LDR slider is moved down past the threshold.
- Clicking the Reset Button silences the alarm and turns the LED OFF.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `raw < DARK_LIMIT` | Checks if the measured light level is below the threshold, indicating dark conditions. |
| `alarm_state = True` | Latches the alarm state to `True` so the alarm continues to sound even if the light is restored. |

## Hardware & Safety Concept: Alarm Latching and Reset Loops
Safety standards require **alarm latching** for critical parameters. Once a threshold is crossed, the alarm must continue to sound even if the parameter returns to safe limits (e.g. the room brightens again). This ensures that temporary hazard events are noticed by operators, and the system can only be silenced by an authorized manual reset.

## Try This! (Challenges)
1. **Dynamic Flash**: Speed up the flashing rate (decrease pulse sleep duration) as the light level drops even lower.
2. **Ambient Brightness Alarm**: Invert the logic to sound the alarm when a bright light is detected in a dark room (closet entry sensor).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly | Threshold set too high | Print LDR values to the terminal to find the ambient light level, then adjust `DARK_LIMIT` in your code. |
| Alarm does not disarm | Reset button pin mismatch | Verify the reset button is wired to GP13 as configured in the code. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [10 - Pico LDR Light Sensor Serial](10-pico-ldr-serial.md)
- [22 - Pico Night Light LDR](22-pico-night-light-ldr.md)
- [33 - Pico Light Buzzer Dark](33-pico-light-buzzer-dark.md)
