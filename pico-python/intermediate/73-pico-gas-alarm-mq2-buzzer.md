# 73 - Pico Gas Alarm MQ2 Buzzer

Build a gas leak detector that sounds a pulsing warning buzzer when an MQ-2 gas sensor detects smoke or combustible gases.

## Goal
Learn how to read digital inputs from gas sensor comparators, implement latching alarm states, and control active warning buzzers in MicroPython.

## What You Will Build
A gas safety alarm:
- **MQ-2 Sensor (GP16)**: Detects smoke/combustible gases (active LOW digital signal).
- **Active Buzzer (GP15)**: Sounds a pulsing siren when gas is detected.
- **Reset Button (GP13)**: Silences the alarm once the air is clean.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MQ-2 Gas Sensor Module | `button` | Yes (represented by push button component) | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Tactile Push Button | `button` | Yes (reset switch) | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MQ-2 Sensor Board | VCC | 5V (VBUS) | Red | MQ sensors require 5V heater power |
| MQ-2 Sensor Board | GND | GND | Black | Ground reference |
| MQ-2 Sensor Board | DO (Digital Out) | GP16 | Blue | Digital gas trigger (LOW on gas) |
| Reset Button | Terminal 1 | GP13 | White | Reset input (reads LOW on press) |
| Reset Button | Terminal 2 | GND | Black | Shorts GP13 to GND |
| Active Buzzer | VCC (+) | GP15 | Red | Alarm output |
| Active Buzzer | GND (−) | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the MQ-2 sensor's VCC to the Pico's 5V VBUS pin. Connect the digital output to GP16, and the reset button to GP13. Connect the active buzzer to GP15 and GND. All grounds are shared.

## Code
```python
from machine import Pin
import utime

gas_sensor = Pin(16, Pin.IN, Pin.PULL_UP)
btn_reset  = Pin(13, Pin.IN, Pin.PULL_UP)
buzzer     = Pin(15, Pin.OUT)

alarm_state = False
buzzer.value(0)

print("Gas safety alarm armed. Sensor heating...")
utime.sleep(2) # Initial sensor warmup delay

while True:
    # 1. Read sensors
    gas_detected  = (gas_sensor.value() == 0) # Active-LOW on gas
    reset_pressed = (btn_reset.value() == 0)  # Active-LOW on press
    
    # 2. Check for trigger (Set)
    if gas_detected and not alarm_state:
        alarm_state = True
        print(">> WARNING: Gas leak detected! Alarm active.")
        
    # 3. Check for disarm (Reset)
    if reset_pressed and alarm_state:
        alarm_state = False
        buzzer.value(0) # Silence immediately
        print(">> Gas alarm reset.")
        utime.sleep(1) # Lockout delay
        
    # 4. Pulsing safety siren
    if alarm_state:
        buzzer.value(1)
        utime.sleep_ms(150)
        buzzer.value(0)
        utime.sleep_ms(150)
    else:
        utime.sleep_ms(50) # Normal polling speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two Push Buttons** (representing gas sensor and reset), and **Active Buzzer** onto the canvas.
2. Connect Gas Sensor to **GP16**, Reset Button to **GP13**, and Buzzer to **GP15**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the Gas Sensor button to trigger the alarm, then click the Reset Button to silence it.

## Expected Output
```
Gas safety alarm armed. Sensor heating...
>> WARNING: Gas leak detected! Alarm active.
>> Gas alarm reset.
```

## Expected Canvas Behavior
- Clicking the Gas Sensor button causes the buzzer component to pulse active continuously.
- Clicking the Reset Button stops the buzzer pulses immediately.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `gas_sensor.value() == 0` | Detects when the gas sensor comparator pulls GP16 LOW, indicating gas leak. |
| `alarm_state = True` | Latches the alarm state to `True` so the alarm continues to sound even if the gas disperses. |

## Hardware & Safety Concept: Sensor Heater Safety
MQ series gas sensors contain an internal **heating element** that warms a tin dioxide semiconductor layer. Under fault conditions, heaters can draw significant current and run hot. In real-world designs, the sensor must be mounted away from flammable materials, and the heater power supply must include overcurrent fuses to prevent thermal runaways.

## Try This! (Challenges)
1. **Exhaust Fan Relay**: Connect an LED/Relay on GP14 that turns ON (simulating a ventilation fan) when the alarm is active.
2. **Visual strobe**: Connect a Red LED on GP12 to flash in sync with the buzzer siren.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly on boot | Sensor heating phase | Allow the sensor to run for a few minutes to heat up and stabilize. |
| Reset button does not disarm | Button pin mismatch | Verify the reset button is wired to GP13 as configured in the code. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [08 - Pico Button Buzzer](08-pico-button-buzzer.md)
- [47 - Pico Smoke Sensor MQ2 Serial](47-pico-smoke-sensor-mq2-serial.md)
- [72 - Pico Flame Alarm Buzzer](72-pico-flame-alarm-buzzer.md)
