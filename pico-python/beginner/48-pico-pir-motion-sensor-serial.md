# 48 - Pico PIR Motion Sensor Serial

Detect motion events using a passive infrared (PIR) sensor and print status logs to the serial monitor.

## Goal
Learn how to configure a digital input pin to read PIR motion signals, handle transition delays, and log event states in MicroPython.

## What You Will Build
A motion detection console:
- **PIR Sensor (GP14)**: Detects moving heat sources (warm bodies).
- **Serial Output**: Prints alerts (e.g. "MOTION DETECTED!" or "Clear") to the terminal in real time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| PIR Motion Sensor | `button` | Yes (represented by push button component) | Yes (PIR module) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| PIR Sensor | VCC (+) | 5V (VBUS) or 3.3V | Red | Power line |
| PIR Sensor | GND (−) | GND | Black | Ground reference |
| PIR Sensor | OUT (Digital) | GP14 | Blue | Digital motion signal (HIGH on motion) |

> **Wiring tip:** Connect the PIR sensor's VCC to 3.3V (or 5V VBUS depending on sensor model), GND to GND, and the digital OUT pin to GP14. In MbedO, the digital trigger is simulated with a button component.

## Code
```python
from machine import Pin
import utime

# PIR OUT pins drive HIGH (3.3V) when motion is active
pir = Pin(14, Pin.IN, Pin.PULL_DOWN)

# Store previous state to detect transition changes
prev_state = 0
pir.value()

print("Motion security console active.")

while True:
    curr_state = pir.value()
    
    # Check for state transition
    if curr_state != prev_state:
        if curr_state == 1:
            print("ALERT: Motion detected! (HIGH)")
        else:
            print("STATUS: Area clear. (LOW)")
        prev_state = curr_state
        
    utime.sleep_ms(100) # Polling update speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Push Button** (representing the PIR sensor) onto the canvas.
2. Connect Button Terminal 1 to **GP14** and Terminal 2 to **3.3V** (since PIR outputs HIGH on motion).
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the button component on the canvas to simulate motion events.

## Expected Output
```
Motion security console active.
ALERT: Motion detected! (HIGH)
STATUS: Area clear. (LOW)
```

## Expected Canvas Behavior
- The serial terminal prints motion alerts when the button component on the canvas is clicked and released.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Pin(14, Pin.IN, Pin.PULL_DOWN)` | Configures GP14 as an input with internal pull-down, defaulting to 0 (LOW) when no motion is present. |
| `curr_state != prev_state` | State transition checker that triggers logs only at the start and end of motion events. |

## Hardware & Safety Concept: PIR Sensor Delay Times
PIR modules contain two onboard trim potentiometers. One potentiometer adjusts **sensitivity** (detection range up to 7 meters), and the other adjusts **time delay** (how long the output stays HIGH after motion stops, adjustable from 3 seconds to 5 minutes). Standard security systems poll the signal continuously but check for state transitions to log unique entry events.

## Try This! (Challenges)
1. **Intrusion Warning LED**: Connect an LED on GP15 that turns ON when motion is detected.
2. **Audio Warning Alarm**: Connect a buzzer on GP13 to sound a warning beep when motion is first detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sensor triggers constantly | Warmup phase active | PIR sensors require a 30 to 60-second warmup period on boot to calibrate to the room's background IR profile. |
| Motion is detected but status lags | Time delay set too long | Rotate the time delay screw counterclockwise on the PIR board to decrease the output hold time. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [03 - Pico Button LED](03-pico-button-led.md)
- [44 - Pico Tilt Switch Serial](44-pico-tilt-switch-serial.md)
