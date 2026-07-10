# 72 - Pico Flame Alarm Buzzer

Build a fire safety alarm that sounds a warning buzzer when a digital flame sensor detects infrared fire wavelengths.

## Goal
Learn how to read digital inputs from flame sensor comparators, implement latching alarm states, and sound pulsing safety sirens in MicroPython.

## What You Will Build
A fire alarm system:
- **Flame Sensor (GP16)**: Detects open flame wavelengths (active LOW digital signal).
- **Active Buzzer (GP15)**: Sounds a loud pulsing alarm when a flame is detected.
- **Reset Button (GP13)**: Silences the alarm once the area is safe.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Flame Sensor Module | `button` | Yes (represented by push button component) | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Tactile Push Button | `button` | Yes (reset switch) | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Flame Sensor Board | VCC | 3.3V (3V3) | Red | Power line |
| Flame Sensor Board | GND | GND | Black | Ground reference |
| Flame Sensor Board | DO (Digital Out) | GP16 | Blue | Digital flame trigger (LOW on fire) |
| Reset Button | Terminal 1 | GP13 | White | Reset input (reads LOW on press) |
| Reset Button | Terminal 2 | GND | Black | Shorts GP13 to GND |
| Active Buzzer | VCC (+) | GP15 | Red | Alarm output |
| Active Buzzer | GND (−) | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the flame sensor's digital output to GP16, and the reset button to GP13. Connect the active buzzer to GP15 and GND. All grounds are shared.

## Code
```python
from machine import Pin
import utime

flame_sensor = Pin(16, Pin.IN, Pin.PULL_UP)
btn_reset    = Pin(13, Pin.IN, Pin.PULL_UP)
buzzer       = Pin(15, Pin.OUT)

alarm_state = False
buzzer.value(0)

print("Flame safety alarm armed.")

while True:
    # 1. Read sensors
    flame_detected = (flame_sensor.value() == 0) # Active-LOW on fire
    reset_pressed  = (btn_reset.value() == 0)    # Active-LOW on press
    
    # 2. Check for trigger (Set)
    if flame_detected and not alarm_state:
        alarm_state = True
        print(">> WARNING: Fire detected! Alarm active.")
        
    # 3. Check for disarm (Reset)
    if reset_pressed and alarm_state:
        alarm_state = False
        buzzer.value(0) # Silence immediately
        print(">> Alarm reset.")
        utime.sleep(1) # Lockout delay
        
    # 4. Pulsing safety siren
    if alarm_state:
        buzzer.value(1)
        utime.sleep_ms(100)
        buzzer.value(0)
        utime.sleep_ms(100)
    else:
        utime.sleep_ms(50) # Normal polling speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two Push Buttons** (representing flame sensor and reset), and **Active Buzzer** onto the canvas.
2. Connect Flame Sensor to **GP16**, Reset Button to **GP13**, and Buzzer to **GP15**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the Flame Sensor button to trigger the alarm, then click the Reset Button to silence it.

## Expected Output
```
Flame safety alarm armed.
>> WARNING: Fire detected! Alarm active.
>> Alarm reset.
```

## Expected Canvas Behavior
- Clicking the Flame Sensor button causes the buzzer component to pulse active continuously.
- Clicking the Reset Button stops the buzzer pulses immediately.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `flame_sensor.value() == 0` | Detects when the flame sensor comparator pulls GP16 LOW, indicating fire. |
| `alarm_state = True` | Latches the alarm state to `True` so the alarm continues to sound even if the flame is extinguished. |

## Hardware & Safety Concept: Alarm Latching and Reset Loops
Fire and gas safety standards require **alarm latching**. Once a hazard is detected, the alarm must continue to sound even if the hazard clears (e.g. the flame goes out). This ensures that building occupants are alerted to the event, and the system can only be silenced by an authorized manual reset.

## Try This! (Challenges)
1. **Fire Suppression Relay**: Connect an LED/Relay on GP14 that turns ON (simulating a sprinkler valve) when the alarm is active.
2. **Visual strobe**: Connect a Red LED on GP12 to flash in sync with the buzzer siren.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly | Ambient light interference | Shield the phototransistor from direct sunlight or incandescent bulbs, which emit infrared light. |
| Reset button does not disarm | Button pin mismatch | Verify the reset button is wired to GP13 as configured in the code. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [08 - Pico Button Buzzer](08-pico-button-buzzer.md)
- [46 - Pico Flame Sensor Serial](46-pico-flame-sensor-serial.md)
- [70 - Pico Tilt Alarm Buzzer](70-pico-tilt-alarm-buzzer.md)
