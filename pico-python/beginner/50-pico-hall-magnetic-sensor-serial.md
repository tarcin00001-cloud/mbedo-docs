# 50 - Pico Hall Magnetic Sensor Serial

Detect magnetic fields using a Hall effect magnetic sensor and print status logs to the serial monitor.

## Goal
Learn how to poll a digital input pin in real time, handle magnetic sensor state transitions, and log magnetic events in MicroPython.

## What You Will Build
A magnetic field monitor:
- **Hall Effect Sensor (GP14)**: Detects the presence of magnetic fields (typically from permanent magnets).
- **Serial Output**: Prints status logs (e.g. "MAGNET DETECTED!" or "No field") to the terminal when a magnet is brought close.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Hall Effect Sensor | `button` | Yes (represented by push button component) | Yes (active-low Hall module) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Hall Sensor | VCC (+) | 3.3V (3V3) or 5V | Red | Power line |
| Hall Sensor | GND (−) | GND | Black | Ground reference |
| Hall Sensor | OUT (Digital) | GP14 | Blue | Digital signal output (LOW on magnet) |

> **Wiring tip:** Connect the Hall sensor's VCC to 3.3V, GND to GND, and OUT to GP14. In MbedO, the digital trigger is simulated with a button component.

## Code
```python
from machine import Pin
import utime

# Digital Hall effect sensors typically output LOW (0V) when a magnetic field is present
hall_sensor = Pin(14, Pin.IN, Pin.PULL_UP)

# Store previous state to detect transitions
prev_state = 1
hall_sensor.value()

print("Hall effect magnetic console active.")

while True:
    curr_state = hall_sensor.value()
    
    # Check for state transition
    if curr_state != prev_state:
        if curr_state == 0:
            print("ALERT: Magnetic field detected! (LOW)")
        else:
            print("STATUS: No magnetic field. (HIGH)")
        prev_state = curr_state
        
    utime.sleep_ms(50) # Polling update speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Push Button** (representing the Hall sensor) onto the canvas.
2. Connect Button Terminal 1 to **GP14** and Terminal 2 to **GND**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the button component on the canvas to simulate magnetic events.

## Expected Output
```
Hall effect magnetic console active.
ALERT: Magnetic field detected! (LOW)
STATUS: No magnetic field. (HIGH)
```

## Expected Canvas Behavior
- The serial terminal prints magnetic alerts when the button component on the canvas is clicked and released.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Pin(14, Pin.IN, Pin.PULL_UP)` | Configures GP14 as an input with internal pull-up, defaulting to 1 (HIGH) when no magnetic field is near. |
| `curr_state != prev_state` | State transition checker that triggers logs only when a magnetic field enters or leaves the detection range. |

## Hardware & Safety Concept: Hall Effect Transducers
Hall effect sensors detect magnetic fields using the Hall effect principle: when a magnetic field passes through a current-carrying conductor, it generates a transverse voltage difference. Digital Hall sensors (latch or switch types) pull their digital output pin to **GND (LOW)** when the magnetic flux density exceeds a trigger threshold. These sensors are widely used in speedometers, brushless motor commutation, and door safety interlock switches.

## Try This! (Challenges)
1. **Safety Interlock LED**: Connect an LED on GP15 that turns ON when a magnet is detected, simulating a closed door.
2. **Alert Buzzer**: Connect a buzzer on GP13 to sound a warning beep when a magnet is detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sensor does not trigger with magnet | Magnet pole reversed | Hall sensors are polar-sensitive (typically responding only to the South pole). Flip the magnet over and try again. |
| Sensor triggers constantly | Loose signal connection | Ensure the OUT pin is connected firmly to GP14 with the internal pull-up enabled. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [03 - Pico Button LED](03-pico-button-led.md)
- [44 - Pico Tilt Switch Serial](44-pico-tilt-switch-serial.md)
- [49 - Pico IR Obstacle Sensor Serial](49-pico-ir-obstacle-sensor-serial.md)
