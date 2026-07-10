# 44 - Pico Tilt Switch Serial

Read state changes from a digital tilt switch sensor and print orientation alerts to the serial monitor.

## Goal
Learn how to poll a digital input pin in real time, detect orientation changes, and print status messages to the serial console in MicroPython.

## What You Will Build
An orientation change monitor:
- **Tilt Switch (GP14)**: Detects physical orientation changes.
- **Serial Output**: Prints orientation status (e.g. "LEVEL" or "TILTED") to the terminal immediately when the sensor is tilted.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Tilt Switch Sensor | `button` | Yes (represented by push button component) | Yes (mercury/ball tilt switch) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Tilt Switch | Pin 1 | GP14 | Blue | Digital input pin (reads LOW when tilted) |
| Tilt Switch | Pin 2 | GND | Black | Shorts GP14 to GND when tilted |

> **Wiring tip:** Tilt switch sensors behave like standard push buttons. By enabling the internal pull-up on GP14, the pin reads HIGH (3.3V) when level, and LOW (0V) when tilted. Connect the switch between GP14 and GND.

## Code
```python
from machine import Pin
import utime

tilt_sensor = Pin(14, Pin.IN, Pin.PULL_UP)

# Store previous state to detect transitions
prev_state = 1
tilt_sensor.value()

print("Tilt sensor console armed. Rotate the sensor!")

while True:
    curr_state = tilt_sensor.value()
    
    # Check for state transition
    if curr_state != prev_state:
        if curr_state == 0:
            print("ALERT: Orientation tilted! (LOW)")
        else:
            print("STATUS: Orientation level. (HIGH)")
        prev_state = curr_state
        
    utime.sleep_ms(50) # Polling update delay
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Push Button** (representing the tilt switch) onto the canvas.
2. Connect Button Terminal 1 to **GP14** and Terminal 2 to **GND**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the button component on the canvas to simulate tilting the sensor.

## Expected Output
```
Tilt sensor console armed. Rotate the sensor!
ALERT: Orientation tilted! (LOW)
STATUS: Orientation level. (HIGH)
```

## Expected Canvas Behavior
- The serial terminal prints orientation alerts when the button component on the canvas is clicked and released.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `curr_state != prev_state` | State transition checker that triggers actions only when the sensor orientation changes. |
| `utime.sleep_ms(50)` | Fast polling delay that provides basic mechanical debouncing. |

## Hardware & Safety Concept: Ball Tilt Sensors
Digital tilt switches contain a tiny conductive metal ball inside a hollow tube. When tilted, the ball rolls to the bottom of the tube and bridges two electrical contacts, completing a circuit. Because the ball can bounce and chatter during motion, software-based transition checks are necessary to prevent rapid false alerts.

## Try This! (Challenges)
1. **Safety Interlock LED**: Connect an LED on GP15 that turns ON when the system is tilted, simulating a safety cutoff.
2. **Alarm Siren**: Connect a buzzer on GP13 to sound a warning beep if the system remains tilted for more than 3 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Orientation triggers repeatedly | Sensor vibration/noise | Increase the polling delay to 100 ms to help filter out mechanical bounce. |
| Sensor status is always LEVEL | Pin connection issue | Verify that the tilt switch is wired correctly to GP14 and that the internal pull-up is enabled. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [03 - Pico Button LED](03-pico-button-led.md)
- [13 - Pico Button Counter](13-pico-button-counter.md)
- [40 - Pico Alarm System Button Buzzer](40-pico-alarm-system-button-buzzer.md)
