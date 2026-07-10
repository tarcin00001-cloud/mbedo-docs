# 70 - Pico Tilt Alarm Buzzer

Build an anti-tamper security alarm that sounds a warning buzzer when a digital tilt switch detects orientation movement.

## Goal
Learn how to read digital tilt switches, implement latching alarm states, and sound pulsing buzzer warnings in MicroPython.

## What You Will Build
An anti-tamper security alarm:
- **Tilt Switch (GP14)**: Detects physical movement/tampering.
- **Active Buzzer (GP15)**: Emits a continuous warning beep when tilted.
- **Reset Button (GP13)**: Silences the alarm and resets the security state.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Tilt Switch Sensor | `button` | Yes (represented by push button component) | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Tactile Push Button | `button` | Yes (reset switch) | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Tilt Switch | Pin 1 | GP14 | Blue | Tilt input (reads LOW when tilted) |
| Tilt Switch | Pin 2 | GND | Black | Shorts GP14 to GND |
| Reset Button | Terminal 1 | GP13 | White | Reset input (reads LOW on press) |
| Reset Button | Terminal 2 | GND | Black | Shorts GP13 to GND |
| Active Buzzer | VCC (+) | GP15 | Red | Alarm output |
| Active Buzzer | GND (−) | GND | Black | Shared return path to ground |

> **Wiring tip:** The tilt switch and reset button connect between their respective GPIO pins (GP14 and GP13) and GND, using internal pull-up resistors. Connect the active buzzer to GP15 and GND.

## Code
```python
from machine import Pin
import utime

tilt_sensor = Pin(14, Pin.IN, Pin.PULL_UP)
btn_reset   = Pin(13, Pin.IN, Pin.PULL_UP)
buzzer      = Pin(15, Pin.OUT)

alarm_state = False
buzzer.value(0)

print("Anti-tamper security system armed.")

while True:
    # 1. Read sensors
    tilt_triggered = (tilt_sensor.value() == 0) # Active-LOW on tilt
    reset_pressed  = (btn_reset.value() == 0)   # Active-LOW on press
    
    # 2. Check for trigger (Set)
    if tilt_triggered and not alarm_state:
        alarm_state = True
        print(">> ALERT: Tampering detected! Alarm active.")
        
    # 3. Check for disarm (Reset)
    if reset_pressed and alarm_state:
        alarm_state = False
        buzzer.value(0) # Silence immediately
        print(">> Alarm disarmed and reset.")
        utime.sleep(1) # Lockout delay
        
    # 4. Pulsing warning sound
    if alarm_state:
        buzzer.value(1)
        utime.sleep_ms(150)
        buzzer.value(0)
        utime.sleep_ms(150)
    else:
        utime.sleep_ms(50) # Normal polling speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two Push Buttons** (representing tilt switch and reset), and **Active Buzzer** onto the canvas.
2. Connect Tilt Switch to **GP14**, Reset Button to **GP13**, and Buzzer to **GP15**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the Tilt Switch button to trigger the alarm, then click the Reset Button to silence it.

## Expected Output
```
Anti-tamper security system armed.
>> ALERT: Tampering detected! Alarm active.
>> Alarm disarmed and reset.
```

## Expected Canvas Behavior
- Clicking the Tilt Switch button causes the buzzer component to pulse active continuously.
- Clicking the Reset Button stops the buzzer pulses immediately.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `tilt_sensor.value() == 0` | Detects when the tilt switch closes, pulling GP14 LOW. |
| `alarm_state = True` | Latches the alarm state to `True` so the alarm continues to sound even if the sensor returns to level. |

## Hardware & Safety Concept: Anti-Tamper Switch Loops
Security control panels and vaults include physical anti-tamper switches. If an intruder attempts to open the housing or move the enclosure, the tamper circuit opens or tilts, immediately triggering a latched alarm state. Latching ensures the alarm continues to sound even if the panel is quickly closed again.

## Try This! (Challenges)
1. **Warning LED**: Connect a Red LED on GP12 that flashes in sync with the buzzer siren.
2. **Tilt Calibration Delay**: Add a 3-second arming delay on boot to allow the operator to place the device down before the security system arms.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers immediately on boot | Sensor orientation | Ensure the tilt switch is placed level. If using a ball tilt switch, mechanical vibrations can cause false alarms. |
| Reset button does not disarm | Button pin mismatch | Verify the reset button is wired to GP13 as configured in the code. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [08 - Pico Button Buzzer](08-pico-button-buzzer.md)
- [44 - Pico Tilt Switch Serial](44-pico-tilt-switch-serial.md)
- [70 - Pico Tilt Alarm Buzzer](70-pico-tilt-alarm-buzzer.md)
