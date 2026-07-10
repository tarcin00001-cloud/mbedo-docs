# 75 - Pico IR Obstacle Avoidance LED

Build a proximity alert indicator that sounds a warning buzzer and turns ON a warning LED when an IR obstacle sensor detects a nearby object.

## Goal
Learn how to read digital inputs from IR obstacle sensors, handle active-low logic levels, and actuate visual/audio warnings in MicroPython.

## What You Will Build
A collision warning system:
- **IR Proximity Sensor (GP14)**: Detects nearby surfaces (active LOW digital signal).
- **LED (GP15)**: Turns ON to provide a visual warning when an obstacle is detected.
- **Active Buzzer (GP13)**: Sounds a warning beep in sync with the LED.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| IR Obstacle Sensor | `button` | Yes (represented by push button component) | Yes |
| LED (any colour) | `led` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| IR Sensor | VCC | 3.3V (3V3) or 5V | Red | Power line |
| IR Sensor | GND | GND | Black | Ground reference |
| IR Sensor | OUT | GP14 | Blue | Digital proximity signal (LOW on obstacle) |
| LED | Anode (+, longer leg) | GP15 | Orange | Warning indicator output pin |
| LED | Cathode (−, shorter leg) | GND | Black | Shared return path to ground |
| 330 Ω Resistor | Either leg | In series with LED Anode | — | Limits LED current |
| Active Buzzer | VCC (+) | GP13 | Red | Warning sound output pin |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the IR sensor output to GP14, LED anode to GP15 through the resistor, and active buzzer VCC to GP13. All ground pins are connected together.

## Code
```python
from machine import Pin
import utime

# IR proximity sensors typically output LOW (0V) when an obstacle is detected
ir_sensor = Pin(14, Pin.IN, Pin.PULL_UP)
warning_led = Pin(15, Pin.OUT)
warning_buz = Pin(13, Pin.OUT)

# Start silent and dark
warning_led.value(0)
warning_buz.value(0)

print("Proximity warning system active.")

while True:
    obstacle_detected = (ir_sensor.value() == 0) # Active-LOW on obstacle
    
    if obstacle_detected:
        warning_led.value(1) # Turn warning LED ON
        warning_buz.value(1) # Turn warning buzzer ON
        print(">> COLLISION WARNING: Obstacle detected!")
        utime.sleep_ms(200)   # Active alarm pulse
        warning_led.value(0)
        warning_buz.value(0)
        utime.sleep_ms(100)
    else:
        warning_led.value(0)
        warning_buz.value(0)
        utime.sleep_ms(50) # Normal polling speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Push Button** (representing IR sensor), **LED**, and **Active Buzzer** onto the canvas.
2. Connect IR Button to **GP14**, LED to **GP15**, and Buzzer to **GP13**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click and hold the button component on the canvas to simulate an obstacle.

## Expected Output
```
Proximity warning system active.
>> COLLISION WARNING: Obstacle detected!
```

## Expected Canvas Behavior
- Clicking the button component causes the LED and buzzer components on the canvas to pulse ON and OFF rapidly.
- Releasing the button silences the alarm and turns the LED OFF.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `ir_sensor.value() == 0` | Detects when the IR sensor's output is pulled LOW, indicating a nearby object is reflecting IR light. |
| `warning_led.value(1); warning_buz.value(1)` | Activates both warning outputs together when an obstacle is detected. |

## Hardware & Safety Concept: Optical Proximity Ranges
Infrared obstacle sensors use an infrared transmitter and receiver pair. Proximity detection range (typically **2 to 30 cm**) is set by adjusting the onboard sensitivity potentiometer. Because ambient light (especially sunlight) contains infrared waves, these sensors can false-trigger outdoors. They are best suited for indoor applications like robot obstacle avoidance.

## Try This! (Challenges)
1. **Latching Hazard Mode**: Modify the code to latch the alarm state ON once triggered, requiring a reset button on GP12 to clear it.
2. **Reverse Alert**: sound the buzzer faster (increase beep frequency) as the object gets closer (simulating parking assist sensors).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly | Sensitivity set too high | Rotate the sensitivity screw counterclockwise on the IR sensor board to decrease the range. |
| Sensor does not detect objects | Direct sunlight interference | Move the sensor away from direct sunlight or bright incandescent lights. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [03 - Pico Button LED](03-pico-button-led.md)
- [49 - Pico IR Obstacle Sensor Serial](49-pico-ir-obstacle-sensor-serial.md)
- [74 - Pico Motion Security PIR Alarm](74-pico-motion-security-pir-alarm.md)
