# 69 - Pico Rain Window Servo

Build a smart home window actuator that automatically closes a window (steers a servo) when rain is detected.

## Goal
Learn how to monitor digital rain sensors, handle state changes, and drive servo motors to physical limits (open/closed angles) in MicroPython.

## What You Will Build
An automatic window closer:
- **Rain Sensor (GP16)**: Detects rain droplets (active LOW digital signal).
- **Servo Motor (GP10)**: Steers to 90 degrees (window open) when dry, and automatically rotates to 0 degrees (window closed) when rain is detected.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Rain Sensor Module | `button` | Yes (represented by push button component) | Yes |
| Servo Motor | `servo` | Yes | Yes (standard SG90) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Rain Sensor Board | VCC (+) | 3.3V (3V3) | Red | Power line |
| Rain Sensor Board | GND (−) | GND | Black | Ground reference |
| Rain Sensor Board | DO (Digital Out) | GP16 | Blue | Digital rain trigger (LOW on rain) |
| Servo Motor | Control (PWM) | GP10 | Orange | PWM control signal |
| Servo Motor | Power (+) | 5V (VBUS) | Red | Servo power supply (5V) |
| Servo Motor | GND (−) | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the rain board's digital output to GP16 (with internal pull-up). Connect the servo control pin to GP10, power to 5V VBUS, and GND to GND.

## Code
```python
from machine import Pin, PWM
import utime

rain_sensor = Pin(16, Pin.IN, Pin.PULL_UP)

# Set up Servo on GP10 (50 Hz PWM)
servo = PWM(Pin(10))
servo.freq(50)

# Servo angles (latch positions)
ANGLE_CLOSE = 0   # Closed position (0 degrees)
ANGLE_OPEN  = 90  # Open position (90 degrees)

MIN_DUTY = 1638 # ~0.5ms (0 degrees)
MAX_DUTY = 8192 # ~2.5ms (180 degrees)

def set_window_angle(angle):
    duty = MIN_DUTY + int(angle * (MAX_DUTY - MIN_DUTY) / 180)
    servo.duty_u16(duty)

# Start state: assume dry and open window
set_window_angle(ANGLE_OPEN)
window_open = True
prev_rain_state = 1 # 1 = Dry (HIGH)

print("Smart window controller armed.")

while True:
    curr_rain_state = rain_sensor.value()
    
    # Check for transition change
    if curr_rain_state != prev_rain_state:
        if curr_rain_state == 0: # Rain detected (LOW)
            window_open = False
            set_window_angle(ANGLE_CLOSE)
            print(">> Rain detected! Closing window.")
        else: # Dried out (HIGH)
            window_open = True
            set_window_angle(ANGLE_OPEN)
            print(">> Weather cleared. Opening window.")
        prev_rain_state = curr_rain_state
        
    utime.sleep_ms(200) # Polling update speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Push Button** (representing the rain sensor), and **Servo Motor** onto the canvas.
2. Connect Button to **GP16** and Servo Control to **GP10**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the button component on the canvas to simulate rain and observe the servo rotating to the closed position.

## Expected Output
```
Smart window controller armed.
>> Rain detected! Closing window.
>> Weather cleared. Opening window.
```

## Expected Canvas Behavior
- The servo motor shaft rotates from 90° to 0° when the button is clicked, and returns to 90° when the button is released.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `curr_rain_state == 0` | Detects a rain event (LOW signal from the sensor board's digital comparator). |
| `set_window_angle(ANGLE_CLOSE)` | Steers the servo to 0 degrees to close the window. |

## Hardware & Safety Concept: Actuator Torque Safety
Motorized windows and skylights use high-torque motors to ensure a tight weather seal when closed. To prevent injuries (pinch hazards) during automated operation, smart window controllers monitor motor current and stop closing immediately if they detect an obstruction (increased current draw), or use physical slip clutches.

## Try This! (Challenges)
1. **Pinch Safety Alarm**: Connect a button on GP14 to simulate a pinch safety sensor. If pressed while the window is closing, reverse the servo immediately to the open position.
2. **Audio Warning Alert**: Connect a buzzer on GP13 to sound a warning beep while the window is closing.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo does not rotate | Insufficient current | Ensure the servo's power (+) pin is connected to the 5V VBUS pin, not 3.3V. |
| Window cycles open and closed rapidly | Sensor bouncing | Add a longer delay (e.g. 5 seconds) after the rain sensor clears before reopening the window, to ensure the weather has stabilized. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [43 - Pico Rain Detector Serial](43-pico-rain-detector-serial.md)
- [59 - Pico Servo Knob](59-pico-servo-knob.md)
