# 87 - Pico Magnetic Security LCD

Build a door-entry security control panel that displays door open/closed status on an I2C LCD and sounds a warning buzzer when a magnetic switch (Hall sensor) triggers.

## Goal
Learn how to read digital inputs from Hall effect sensors, handle active-low logic levels, output status text and warning alerts to an I2C character LCD, and implement latching entry alarms in MicroPython.

## What You Will Build
A door-entry console:
- **Hall Effect Sensor (GP14)**: Detects a nearby magnet (active LOW when magnet is present).
- **Active Buzzer (GP15)**: Sounds a pulsing siren when the door opens (magnet is removed, signal goes HIGH).
- **Reset Button (GP13)**: Silences the alarm and resets the security state.
- **I2C 16x2 LCD (GP4, GP5)**: Displays the door state ("DOOR OPEN! ALARM" or "DOOR CLOSED") and active alarm messages.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Hall Effect Sensor | `button` | Yes (represented by push button component) | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Tactile Push Button | `button` | Yes (reset switch) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Hall Sensor | VCC | 3.3V (3V3) | Red | Power line |
| Hall Sensor | GND | GND | Black | Ground reference |
| Hall Sensor | OUT | GP14 | Blue | Digital signal output (LOW on magnet present) |
| Reset Button | Terminal 1 | GP13 | White | Reset input (reads LOW on press) |
| Reset Button | Terminal 2 | GND | Black | Shorts GP13 to GND |
| Active Buzzer | VCC (+) | GP15 | Red | Alarm output |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the Hall sensor output to GP14, reset button to GP13, and active buzzer VCC to GP15. Connect the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime
from machine_lcd import I2cLcd

# Hall sensors output LOW (0) when a magnet is present (door closed)
# Output goes HIGH (1) when the magnet is removed (door open)
door_sensor = Pin(14, Pin.IN, Pin.PULL_UP)
btn_reset   = Pin(13, Pin.IN, Pin.PULL_UP)
buzzer      = Pin(15, Pin.OUT)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

alarm_state = False
buzzer.value(0)

lcd.clear()
lcd.putstr("Access Panel")
lcd.move_to(0, 1)
lcd.putstr("System Arming...")
utime.sleep(2)

# Initial LCD update
lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("SECURE ZONE")
lcd.move_to(0, 1)
lcd.putstr("Door: CLOSED")

while True:
    # Door is open if sensor output is HIGH (magnet removed)
    door_opened   = (door_sensor.value() == 1)
    reset_pressed = (btn_reset.value() == 0)
    
    # 2. Check for trigger (Set)
    if door_opened and not alarm_state:
        alarm_state = True
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("! ENTRY ALERT !")
        lcd.move_to(0, 1)
        lcd.putstr("DOOR OPENED")
        print(">> SECURITY ALERT: Door opened! Alarm active.")
        
    # 3. Check for disarm (Reset)
    if reset_pressed and alarm_state:
        alarm_state = False
        buzzer.value(0) # Silence immediately
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("SECURE ZONE")
        lcd.move_to(0, 1)
        lcd.putstr("Door: CLOSED")
        print(">> Security alarm disarmed.")
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
1. Drag the **Raspberry Pi Pico**, **two Push Buttons** (representing door sensor and reset), **Active Buzzer**, and **I2C LCD** onto the canvas.
2. Connect Door Sensor Button to **GP14**, Reset Button to **GP13**, Buzzer to **GP15**, and LCD to **GP4/GP5**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the Door Sensor button (simulating door opening/magnet removed) to trigger the alarm. Click Reset to silence it.

## Expected Output
```
Magnetic security system armed.
>> SECURITY ALERT: Door opened! Alarm active.
>> Security alarm disarmed.
```
(On screen: "SECURE ZONE / Door: CLOSED" initially; changing to "! ENTRY ALERT ! / DOOR OPENED" when triggered.)

## Expected Canvas Behavior
- Clicking the Door Sensor button component causes the buzzer component to pulse active and the LCD screen to update with warning text.
- Clicking the Reset Button returns the LCD screen to the safe message.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `door_sensor.value() == 1` | Detects when the magnet is removed (door opens), causing the pull-up to pull the pin HIGH. |
| `alarm_state = True` | Latches the alarm state ON so the siren continues to sound even if the door is closed again. |

## Hardware & Safety Concept: Magnetic Reed Switches
Door and window security contacts typically use **magnetic reed switches**. These switches consist of two metal reeds inside a sealed glass tube. When a magnet (mounted on the door frame) is close, the reeds touch and complete the circuit. If the door opens, the magnet moves away, the reeds separate, and the circuit breaks. Reed switches are reliable, require no power, and last for millions of cycles.

## Try This! (Challenges)
1. **Indicator Spotlight**: Connect a relay on GP12 to turn ON a security light when the alarm is active.
2. **Visual Status LED**: Connect a Green LED on GP11 that glows when the door is safely closed (magnet present).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly | Magnet pole reversed | Hall sensors are polar-sensitive. Flip the magnet over to ensure the correct pole faces the sensor. |
| Reset button does not disarm | Button pin mismatch | Verify the reset button is wired to GP13 as configured in the code. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [76 - Pico Magnetic Security Alarm](76-pico-magnetic-security-alarm.md)
- [85 - Pico Motion Security PIR LCD](85-pico-motion-security-pir-lcd.md)
