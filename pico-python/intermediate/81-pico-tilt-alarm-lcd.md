# 81 - Pico Tilt Alarm LCD

Build an anti-tamper security panel that displays orientation safety status on an I2C character LCD and sounds a warning buzzer when tilted.

## Goal
Learn how to read digital tilt switches, manage latching alarm states, output alert messages to an I2C character LCD, and control warning buzzers in MicroPython.

## What You Will Build
An anti-tamper security console:
- **Tilt Switch (GP14)**: Detects physical tampering (active LOW).
- **Active Buzzer (GP15)**: Emits warning beeps when triggered.
- **Reset Button (GP13)**: Disarms the alarm and resets the status.
- **I2C 16x2 LCD (GP4, GP5)**: Displays the system state ("SYSTEM SAFE" or "TAMPER ALERT!").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Tilt Switch Sensor | `button` | Yes (represented by push button component) | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Tactile Push Button | `button` | Yes (reset switch) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Tilt Switch | Pin 1 | GP14 | Blue | Tilt input (reads LOW when tilted) |
| Tilt Switch | Pin 2 | GND | Black | Shorts GP14 to GND |
| Reset Button | Terminal 1 | GP13 | White | Reset input (reads LOW on press) |
| Reset Button | Terminal 2 | GND | Black | Shorts GP13 to GND |
| Active Buzzer | VCC (+) | GP15 | Red | Alarm output |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the tilt switch output to GP14, reset button to GP13, and active buzzer VCC to GP15. Connect the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime
from machine_lcd import I2cLcd

tilt_sensor = Pin(14, Pin.IN, Pin.PULL_UP)
btn_reset   = Pin(13, Pin.IN, Pin.PULL_UP)
buzzer      = Pin(15, Pin.OUT)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

alarm_state = False
buzzer.value(0)

lcd.clear()
lcd.putstr("Tamper Panel")
lcd.move_to(0, 1)
lcd.putstr("System Arming...")
utime.sleep(2)

# Initial screen setup
lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("SYSTEM ARMED")
lcd.move_to(0, 1)
lcd.putstr("Status: SAFE")

while True:
    tilt_triggered = (tilt_sensor.value() == 0) # Active-LOW on tilt
    reset_pressed  = (btn_reset.value() == 0)   # Active-LOW on press
    
    # 2. Check for trigger (Set)
    if tilt_triggered and not alarm_state:
        alarm_state = True
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("! TAMPER ALERT !")
        lcd.move_to(0, 1)
        lcd.putstr("ALARM ACTIVE")
        print(">> SECURITY ALERT: Tampering detected!")
        
    # 3. Check for disarm (Reset)
    if reset_pressed and alarm_state:
        alarm_state = False
        buzzer.value(0) # Silence immediately
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("SYSTEM DISARMED")
        lcd.move_to(0, 1)
        lcd.putstr("Status: SAFE")
        print(">> Security disarmed.")
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
1. Drag the **Raspberry Pi Pico**, **two Push Buttons** (representing tilt switch and reset), **Active Buzzer**, and **I2C LCD** onto the canvas.
2. Connect Tilt Switch to **GP14**, Reset Button to **GP13**, Buzzer to **GP15**, and LCD to **GP4/GP5**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the Tilt Switch button to trigger the alarm, then click the Reset Button to silence it.

## Expected Output
```
>> SECURITY ALERT: Tampering detected!
>> Security disarmed.
```
(On screen: "SYSTEM ARMED / Status: SAFE" initially; changing to "! TAMPER ALERT ! / ALARM ACTIVE" when triggered.)

## Expected Canvas Behavior
- Clicking the Tilt Switch button component causes the buzzer component to pulse active and the LCD screen to update with warning text.
- Clicking the Reset Button returns the LCD screen to the safe message.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `tilt_sensor.value() == 0` | Checks if the tilt sensor has closed, pulling GP14 LOW. |
| `lcd.putstr("! TAMPER ALERT !")` | Renders a high-priority warning message on the LCD display buffer. |

## Hardware & Safety Concept: Industrial Alarm Displays
Safety control systems require both acoustic alarms and visual status displays to ensure safety messages are delivered clearly. Even if the buzzer buzzer fails or is muffled, the LCD display continues to provide active alert messages, keeping operators informed of the status.

## Try This! (Challenges)
1. **Visual strobe**: Connect a Red LED on GP12 and flash it in sync with the buzzer siren.
2. **Auto-Arming delay**: Add a 5-second countdown on the LCD screen before the security system arms on boot, allowing the operator to place the device down.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers immediately on boot | Sensor orientation | Ensure the tilt switch is placed level. If using a ball tilt switch, mechanical vibrations can cause false alarms. |
| Reset button does not disarm | Button pin mismatch | Verify the reset button is wired to GP13 as configured in the code. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [70 - Pico Tilt Alarm Buzzer](70-pico-tilt-alarm-buzzer.md)
