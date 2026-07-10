# 80 - Pico Rain Window LCD

Build an automated skylight/window control panel that displays storm weather warnings on an I2C LCD and closes a servo-actuated window when rain is detected.

## Goal
Learn how to monitor digital rain sensors, handle state changes, output status text and warning alerts to an I2C character LCD, and drive servo motors to physical limits in MicroPython.

## What You Will Build
A smart window dashboard:
- **Rain Sensor (GP16)**: Detects rain droplets (active LOW digital signal).
- **Servo Motor (GP10)**: Steers to 90 degrees (window open) when dry, and automatically rotates to 0 degrees (window closed) when rain is detected.
- **I2C 16x2 LCD (GP4, GP5)**: Displays active weather status ("DRY - OPEN" or "RAIN - CLOSED") and visual alerts.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Rain Sensor Module | `button` | Yes (represented by push button component) | Yes |
| Servo Motor | `servo` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Rain Sensor Board | VCC | 3.3V (3V3) | Red | Power line |
| Rain Sensor Board | GND | GND | Black | Ground reference |
| Rain Sensor Board | DO (Digital Out) | GP16 | Blue | Digital rain trigger (LOW on rain) |
| Servo Motor | Control (PWM) | GP10 | Orange | PWM control signal |
| Servo Motor | Power (+) | 5V (VBUS) | Red | Servo power supply (5V) |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the rain sensor output to GP16 (with internal pull-up). Connect the servo control pin to GP10, power to 5V VBUS, and GND to GND. Connect the LCD to the GP4/GP5 pins. All ground connections are shared.

## Code
```python
from machine import Pin, PWM, I2C
import utime
from machine_lcd import I2cLcd

rain_sensor = Pin(16, Pin.IN, Pin.PULL_UP)

# Set up Servo on GP10 (50 Hz PWM)
servo = PWM(Pin(10))
servo.freq(50)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Servo angles (latch positions)
ANGLE_CLOSE = 0   # Closed position (0 degrees)
ANGLE_OPEN  = 90  # Open position (90 degrees)

# Servo duty cycle bounds
MIN_DUTY = 1638
MAX_DUTY = 8192

def set_window_angle(angle):
    duty = MIN_DUTY + int(angle * (MAX_DUTY - MIN_DUTY) / 180)
    servo.duty_u16(duty)

# Start state: assume dry and open window
set_window_angle(ANGLE_OPEN)
window_open = True
prev_rain_state = 1 # 1 = Dry (HIGH)

lcd.clear()
lcd.putstr("Skylight Panel")
lcd.move_to(0, 1)
lcd.putstr("Initializing...")
utime.sleep(1)

# Initial screen update
lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("Weather: DRY")
lcd.move_to(0, 1)
lcd.putstr("Window : OPEN")

while True:
    curr_rain_state = rain_sensor.value()
    
    # Check for transition change
    if curr_rain_state != prev_rain_state:
        lcd.clear()
        if curr_rain_state == 0: # Rain detected (LOW)
            window_open = False
            set_window_angle(ANGLE_CLOSE)
            lcd.move_to(0, 0)
            lcd.putstr("Weather: RAIN!")
            lcd.move_to(0, 1)
            lcd.putstr("Window : CLOSED")
            print(">> Rain detected! Closing window.")
        else: # Dried out (HIGH)
            window_open = True
            set_window_angle(ANGLE_OPEN)
            lcd.move_to(0, 0)
            lcd.putstr("Weather: DRY")
            lcd.move_to(0, 1)
            lcd.putstr("Window : OPEN")
            print(">> Weather cleared. Opening window.")
        prev_rain_state = curr_rain_state
        
    utime.sleep_ms(200) # Polling update speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Push Button** (representing rain sensor), **Servo Motor**, and **I2C LCD** onto the canvas.
2. Connect Button to **GP16**, Servo Control to **GP10**, and LCD to **GP4/GP5**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the button component on the canvas to simulate rain and observe the servo and LCD display update.

## Expected Output
```
>> Rain detected! Closing window.
>> Weather cleared. Opening window.
```
(On screen: "Weather: RAIN!" and "Window: CLOSED" when rain is active, and "Weather: DRY" and "Window: OPEN" when dry.)

## Expected Canvas Behavior
- The servo motor shaft rotates from 90° to 0° and the LCD text updates to show "RAIN! / CLOSED" when the rain sensor button is clicked.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `curr_rain_state == 0` | Detects a rain event (LOW signal from the sensor board's digital comparator). |
| `set_window_angle(ANGLE_CLOSE)` | Steers the servo to 0 degrees to close the window. |

## Hardware & Safety Concept: Actuator Torque Safety
Motorized windows and skylights use high-torque motors to ensure a tight weather seal when closed. To prevent injuries (pinch hazards) during automated operation, smart window controllers monitor motor current and stop closing immediately if they detect an obstruction (increased current draw), or use physical slip clutches.

## Try This! (Challenges)
1. **Pinch Safety Switch**: Connect a button on GP14 to simulate a pinch safety sensor. If pressed while the window is closing, reverse the servo immediately to the open position and display "SAFETY BLOCK" on the LCD.
2. **Audio Warning Alert**: Connect a buzzer on GP13 to sound a warning beep while the window is closing.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo does not rotate | Insufficient current | Ensure the servo's power (+) pin is connected to the 5V VBUS pin, not 3.3V. |
| LCD freezes or displays random blocks | Servo electrical noise | Add a decoupling capacitor across the power rails near the servo to filter out voltage drops. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [43 - Pico Rain Detector Serial](43-pico-rain-detector-serial.md)
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [59 - Pico Servo Knob](59-pico-servo-knob.md)
- [69 - Pico Rain Window Servo](69-pico-rain-window-servo.md)
