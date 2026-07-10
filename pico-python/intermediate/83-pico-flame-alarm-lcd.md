# 83 - Pico Flame Alarm LCD

Build a fire safety console that displays active fire status warnings on an I2C LCD and sounds a pulsing warning buzzer when a digital flame sensor triggers.

## Goal
Learn how to read digital inputs from flame sensor comparators, manage latching alarm states, output alert messages to an I2C character LCD, and control warning sirens in MicroPython.

## What You Will Build
A fire safety console:
- **Flame Sensor (GP16)**: Detects open flame wavelengths (active LOW digital signal).
- **Active Buzzer (GP15)**: Sounds a loud pulsing alarm when a flame is detected.
- **Reset Button (GP13)**: Silences the alarm once the area is safe.
- **I2C 16x2 LCD (GP4, GP5)**: Displays the system state ("SYSTEM SAFE" or "FIRE DETECTED!").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Flame Sensor Module | `button` | Yes (represented by push button component) | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Tactile Push Button | `button` | Yes (reset switch) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Flame Sensor Board | VCC | 3.3V (3V3) | Red | Power line |
| Flame Sensor Board | GND | GND | Black | Ground reference |
| Flame Sensor Board | DO (Digital Out) | GP16 | Blue | Digital flame trigger (LOW on fire) |
| Reset Button | Terminal 1 | GP13 | White | Reset input (reads LOW on press) |
| Reset Button | Terminal 2 | GND | Black | Shorts GP13 to GND |
| Active Buzzer | VCC (+) | GP15 | Red | Alarm output |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the flame sensor output to GP16, reset button to GP13, and active buzzer VCC to GP15. Connect the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime
from machine_lcd import I2cLcd

flame_sensor = Pin(16, Pin.IN, Pin.PULL_UP)
btn_reset    = Pin(13, Pin.IN, Pin.PULL_UP)
buzzer       = Pin(15, Pin.OUT)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

alarm_state = False
buzzer.value(0)

lcd.clear()
lcd.putstr("Fire Safety Node")
lcd.move_to(0, 1)
lcd.putstr("Console Armed...")
utime.sleep(2)

# Initial LCD update
lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("SYSTEM ARMED")
lcd.move_to(0, 1)
lcd.putstr("Status: SAFE")

while True:
    flame_detected = (flame_sensor.value() == 0) # Active-LOW on fire
    reset_pressed  = (btn_reset.value() == 0)    # Active-LOW on press
    
    # 2. Check for trigger (Set)
    if flame_detected and not alarm_state:
        alarm_state = True
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("!! FIRE ALERT !!")
        lcd.move_to(0, 1)
        lcd.putstr("EVACUATE AREA")
        print(">> WARNING: Fire detected! Alarm active.")
        
    # 3. Check for disarm (Reset)
    if reset_pressed and alarm_state:
        alarm_state = False
        buzzer.value(0) # Silence immediately
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("SYSTEM ARMED")
        lcd.move_to(0, 1)
        lcd.putstr("Status: SAFE")
        print(">> Alarm reset.")
        utime.sleep(1) # Lockout delay
        
    # 4. Pulsing safety siren
    if alarm_state:
        buzzer.value(1)
        utime.sleep_ms(100)
        buzzer.value(0)
        utime.sleep_ms(100)
    else:
        utime.sleep_ms(50) # Normal polling speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two Push Buttons** (representing flame sensor and reset), **Active Buzzer**, and **I2C LCD** onto the canvas.
2. Connect Flame Sensor to **GP16**, Reset Button to **GP13**, and Buzzer to **GP15**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the Flame Sensor button to trigger the alarm, then click the Reset Button to silence it.

## Expected Output
```
Fire safety alarm armed.
>> WARNING: Fire detected! Alarm active.
>> Alarm reset.
```
(On screen: "SYSTEM ARMED / Status: SAFE" initially; changing to "!! FIRE ALERT !! / EVACUATE AREA" when triggered.)

## Expected Canvas Behavior
- Clicking the Flame Sensor button causes the buzzer component to pulse active and the LCD screen to update with warning text.
- Clicking the Reset Button returns the LCD screen to the safe message.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `flame_sensor.value() == 0` | Detects when the flame sensor comparator pulls GP16 LOW, indicating fire. |
| `lcd.putstr("!! FIRE ALERT !!")` | Renders a high-priority warning message on the LCD display buffer. |

## Hardware & Safety Concept: Fire Alarm Strobe Synces
Fire safety consoles combine acoustic sounders with high-contrast text and flashing strobes. In loud environments or for hearing-impaired users, audio warnings can go unnoticed. Adding prominent visual alerts to LCD screens and flashing status LEDs provides a secondary path of warning, improving safety.

## Try This! (Challenges)
1. **Fire Suppression Relay**: Connect an LED/Relay on GP14 that turns ON (simulating a sprinkler valve) when the alarm is active.
2. **Visual strobe**: Connect a Red LED on GP12 to flash in sync with the buzzer siren.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly | Ambient light interference | Shield the phototransistor from direct sunlight or incandescent bulbs, which emit infrared light. |
| Reset button does not disarm | Button pin mismatch | Verify the reset button is wired to GP13 as configured in the code. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [72 - Pico Flame Alarm Buzzer](72-pico-flame-alarm-buzzer.md)
- [81 - Pico Tilt Alarm LCD](81-pico-tilt-alarm-lcd.md)
