# 49 - Pico IR Obstacle Sensor Serial

Detect close obstacles using an infrared (IR) proximity sensor and print status logs to the serial monitor.

## Goal
Learn how to poll a digital input pin in real time, handle active-low sensor outputs, and log proximity events in MicroPython.

## What You Will Build
An proximity detection interface:
- **IR Obstacle Sensor (GP14)**: Detects nearby surfaces (typically within 2-30 cm).
- **Serial Output**: Prints status logs (e.g. "OBSTACLE DETECTED!" or "Clear") to the terminal when an object approaches.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| IR Obstacle Sensor | `button` | Yes (represented by push button component) | Yes (active-low IR sensor) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| IR Sensor | VCC (+) | 3.3V (3V3) or 5V | Red | Power line |
| IR Sensor | GND (−) | GND | Black | Ground reference |
| IR Sensor | OUT (Digital) | GP14 | Blue | Digital signal output (LOW on obstacle) |

> **Wiring tip:** Connect the IR sensor's VCC to 3.3V, GND to GND, and OUT to GP14. In MbedO, the digital trigger is simulated with a button component.

## Code
```python
from machine import Pin
import utime

# IR proximity sensors typically output LOW (0V) when an obstacle is detected
ir_sensor = Pin(14, Pin.IN, Pin.PULL_UP)

# Store previous state to detect transitions
prev_state = 1
ir_sensor.value()

print("IR proximity console active.")

while True:
    curr_state = ir_sensor.value()
    
    # Check for state transition
    if curr_state != prev_state:
        if curr_state == 0:
            print("ALERT: Obstacle detected! (LOW)")
        else:
            print("STATUS: Path clear. (HIGH)")
        prev_state = curr_state
        
    utime.sleep_ms(50) # Polling update speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Push Button** (representing the IR sensor) onto the canvas.
2. Connect Button Terminal 1 to **GP14** and Terminal 2 to **GND**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the button component on the canvas to simulate obstacle events.

## Expected Output
```
IR proximity console active.
ALERT: Obstacle detected! (LOW)
STATUS: Path clear. (HIGH)
```

## Expected Canvas Behavior
- The serial terminal prints proximity alerts when the button component on the canvas is clicked and released.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Pin(14, Pin.IN, Pin.PULL_UP)` | Configures GP14 as an input with internal pull-up, defaulting to 1 (HIGH) when no obstacle is near. |
| `curr_state != prev_state` | State transition checker that triggers logs only when an obstacle enters or leaves the detection range. |

## Hardware & Safety Concept: Active-Low IR Proximity Sensors
IR obstacle sensors contain an **IR transmitter LED** (which emits infrared light) and an **IR receiver photodiode** (which detects reflected IR light). When an object approaches, light is reflected back to the receiver, and the sensor board's onboard comparator pulls the digital output pin (OUT) to **GND (LOW)**. Proximity sensors use active-low logic because pulling a pin to ground is electrically less susceptible to noise than driving it high.

## Try This! (Challenges)
1. **Collision Warning LED**: Connect an LED on GP15 that turns ON when an obstacle is detected.
2. **Reverse Safety Beeper**: Connect a buzzer on GP13 to sound a warning beep when an obstacle is detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sensor triggers constantly | Threshold set too sensitive | On real hardware, rotate the onboard potentiometer screw on the sensor breakout board to adjust the detection range. |
| Sensor does not detect objects | Ambient light interference | Sunlight contains high levels of infrared light. Test the sensor indoors, away from direct sunlight. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [03 - Pico Button LED](03-pico-button-led.md)
- [44 - Pico Tilt Switch Serial](44-pico-tilt-switch-serial.md)
- [48 - Pico PIR Motion Sensor Serial](48-pico-pir-motion-sensor-serial.md)
