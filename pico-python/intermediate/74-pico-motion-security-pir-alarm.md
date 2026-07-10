# 74 - Pico Motion Security PIR Alarm

Build a home security system that sounds a warning buzzer and turns ON a relay light when a PIR motion sensor detects movement.

## Goal
Learn how to monitor digital PIR sensors, handle state changes, control relays, and implement latching alarm states in MicroPython.

## What You Will Build
An intrusion detector alarm:
- **PIR Sensor (GP14)**: Detects motion (HIGH output on detection).
- **Active Buzzer (GP15)**: Sounds a pulsing siren when motion is detected.
- **Relay Module (GP12)**: Turns ON a high-power security spotlight when motion is detected.
- **Reset Button (GP13)**: Silences the alarm and resets the spotlight.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| PIR Motion Sensor | `button` | Yes (represented by push button component) | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| Tactile Push Button | `button` | Yes (reset switch) | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| PIR Sensor | VCC | 5V (VBUS) or 3.3V | Red | Power line |
| PIR Sensor | GND | GND | Black | Ground reference |
| PIR Sensor | OUT | GP14 | Blue | Digital signal output (HIGH on motion) |
| Reset Button | Terminal 1 | GP13 | White | Reset input (reads LOW on press) |
| Reset Button | Terminal 2 | GND | Black | Shorts GP13 to GND |
| Active Buzzer | VCC (+) | GP15 | Red | Alarm output |
| Relay Module | IN | GP12 | Orange | Spotlight relay control |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the PIR output to GP14, reset button to GP13, buzzer to GP15, and relay input to GP12. All grounds are shared.

## Code
```python
from machine import Pin
import utime

# PIR OUT pins drive HIGH (3.3V) when motion is active
pir        = Pin(14, Pin.IN, Pin.PULL_DOWN)
btn_reset  = Pin(13, Pin.IN, Pin.PULL_UP)
buzzer     = Pin(15, Pin.OUT)
light_re   = Pin(12, Pin.OUT)

alarm_state = False
buzzer.value(0)
light_re.value(0)

print("Motion security alarm armed. Calibrating...")
utime.sleep(2) # Allow PIR to stabilize

while True:
    # 1. Read sensors
    motion_detected = (pir.value() == 1)       # Active-HIGH on motion
    reset_pressed   = (btn_reset.value() == 0) # Active-LOW on press
    
    # 2. Check for trigger (Set)
    if motion_detected and not alarm_state:
        alarm_state = True
        light_re.value(1) # Turn spotlight ON
        print(">> INTRUSION ALERT! Motion detected in secure zone.")
        
    # 3. Check for disarm (Reset)
    if reset_pressed and alarm_state:
        alarm_state = False
        buzzer.value(0)   # Silence immediately
        light_re.value(0) # Turn spotlight OFF
        print(">> Security alarm reset.")
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
1. Drag the **Raspberry Pi Pico**, **two Push Buttons** (representing PIR and reset), **Active Buzzer**, and **Relay** onto the canvas.
2. Connect PIR Button to **GP14**, Reset Button to **GP13**, Buzzer to **GP15**, and Relay to **GP12**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the PIR Button to trigger the alarm, then click the Reset Button to silence it.

## Expected Output
```
Motion security alarm armed. Calibrating...
>> INTRUSION ALERT! Motion detected in secure zone.
>> Security alarm reset.
```

## Expected Canvas Behavior
- Clicking the PIR Button causes the buzzer component to pulse active and the relay to close.
- Clicking the Reset Button stops the buzzer and opens the relay.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `pir.value() == 1` | Detects when the PIR sensor outputs a HIGH signal, indicating movement. |
| `light_re.value(1)` | Drives GP12 HIGH to close the relay contacts, activating the spotlight. |

## Hardware & Safety Concept: Sensor Warmup Profiles
PIR sensors contain dual pyroelectric infrared sensor chambers that measure ambient heat profiles. When powered ON, the chambers must calibrate to the room's baseline infrared profile. During this **warmup period** (typically 30 to 60 seconds), the sensor's output may trigger randomly. Safety systems ignore sensor outputs during this initialization window.

## Try This! (Challenges)
1. **Auto-Off Timer**: Modify the code so that the alarm and spotlight turn OFF automatically if no new motion is detected for 10 seconds.
2. **Visual strobe**: Connect a Red LED on GP11 to flash in sync with the buzzer siren.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly on boot | Calibration phase active | Wait 30 seconds after powering on for the PIR sensor to calibrate. |
| Reset button does not disarm | Button pin mismatch | Verify the reset button is wired to GP13 as configured in the code. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [08 - Pico Button Buzzer](08-pico-button-buzzer.md)
- [48 - Pico PIR Motion Sensor Serial](48-pico-pir-motion-sensor-serial.md)
- [62 - Pico Relay Button](62-pico-relay-button.md)
- [70 - Pico Tilt Alarm Buzzer](70-pico-tilt-alarm-buzzer.md)
