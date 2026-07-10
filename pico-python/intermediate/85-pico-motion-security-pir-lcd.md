# 85 - Pico Motion Security PIR LCD

Build a home security console that displays intrusion event logs on an I2C LCD, sounds a warning buzzer, and closes a security shutter relay when a PIR motion sensor triggers.

## Goal
Learn how to monitor digital PIR sensors, handle state changes, output status text and event logs to an I2C character LCD, control relays, and manage latching alarm states in MicroPython.

## What You Will Build
An intrusion monitoring console:
- **PIR Sensor (GP14)**: Detects motion (HIGH output on detection).
- **Active Buzzer (GP15)**: Sounds a pulsing siren when motion is detected.
- **Relay Module (GP12)**: Turns ON a high-power security spotlight when motion is detected.
- **Reset Button (GP13)**: Silences the alarm and resets the spotlight.
- **I2C 16x2 LCD (GP4, GP5)**: Displays active weather status ("INTRUDER ALERT!" or "SECURE ZONE") and visual alerts.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| PIR Motion Sensor | `button` | Yes (represented by push button component) | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| Tactile Push Button | `button` | Yes (reset switch) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| PIR Sensor | VCC | 5V (VBUS) or 3.3V | Red | Power line |
| PIR Sensor | GND | GND | Black | Ground reference |
| PIR Sensor | OUT | GP14 | Blue | Digital signal output (HIGH on motion) |
| Reset Button | Terminal 1 | GP13 | White | Reset input (reads LOW on press) |
| Reset Button | Terminal 2 | GND | Black | Shorts GP13 to GND |
| Active Buzzer | VCC (+) | GP15 | Red | Alarm output |
| Relay Module | IN | GP12 | Orange | Spotlight relay control |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the PIR output to GP14, reset button to GP13, buzzer to GP15, and relay input to GP12. Connect the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime
from machine_lcd import I2cLcd

# PIR OUT pins drive HIGH (3.3V) when motion is active
pir        = Pin(14, Pin.IN, Pin.PULL_DOWN)
btn_reset  = Pin(13, Pin.IN, Pin.PULL_UP)
buzzer     = Pin(15, Pin.OUT)
light_re   = Pin(12, Pin.OUT)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

alarm_state = False
buzzer.value(0)
light_re.value(0)

lcd.clear()
lcd.putstr("Security Console")
lcd.move_to(0, 1)
lcd.putstr("Stabilizing PIR...")
utime.sleep(2) # Allow PIR to stabilize

# Initial LCD update
lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("SECURE ZONE")
lcd.move_to(0, 1)
lcd.putstr("Status: READY")

while True:
    # 1. Read sensors
    motion_detected = (pir.value() == 1)       # Active-HIGH on motion
    reset_pressed   = (btn_reset.value() == 0) # Active-LOW on press
    
    # 2. Check for trigger (Set)
    if motion_detected and not alarm_state:
        alarm_state = True
        light_re.value(1) # Turn spotlight ON
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("!INTRUDER ALERT!")
        lcd.move_to(0, 1)
        lcd.putstr("LIGHTS ACTIVE")
        print(">> INTRUSION ALERT! Motion detected in secure zone.")
        
    # 3. Check for disarm (Reset)
    if reset_pressed and alarm_state:
        alarm_state = False
        buzzer.value(0)   # Silence immediately
        light_re.value(0) # Turn spotlight OFF
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("SECURE ZONE")
        lcd.move_to(0, 1)
        lcd.putstr("Status: READY")
        print(">> Security alarm reset.")
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
1. Drag the **Raspberry Pi Pico**, **two Push Buttons** (representing PIR and reset), **Active Buzzer**, **Relay**, and **I2C LCD** onto the canvas.
2. Connect PIR Button to **GP14**, Reset Button to **GP13**, Buzzer to **GP15**, Relay to **GP12**, and LCD to **GP4/GP5**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the PIR Button to trigger the alarm, then click the Reset Button to silence it.

## Expected Output
```
Motion security alarm armed. Calibrating...
>> INTRUSION ALERT! Motion detected in secure zone.
>> Security alarm reset.
```
(On screen: "SECURE ZONE / Status: READY" initially; changing to "!INTRUDER ALERT! / LIGHTS ACTIVE" when triggered.)

## Expected Canvas Behavior
- Clicking the PIR Button causes the buzzer component to pulse active, the relay to close, and the LCD screen to update with warning text.
- Clicking the Reset Button returns the LCD screen to the safe message.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `pir.value() == 1` | Detects when the PIR sensor outputs a HIGH signal, indicating movement. |
| `light_re.value(1)` | Drives GP12 HIGH to close the relay contacts, activating the spotlight. |

## Hardware & Safety Concept: Sensor Warmup Profiles
PIR sensors contain dual pyroelectric infrared sensor chambers that measure ambient heat profiles. When powered ON, the chambers must calibrate to the room's baseline infrared profile. During this **warmup period** (typically 30 to 60 seconds), the sensor's output may trigger randomly. Safety systems ignore sensor outputs during this initialization window.

## Try This! (Challenges)
1. **Auto-Off Timer**: Modify the code so that the alarm and spotlight turn OFF automatically if no new motion is detected for 10 seconds.
2. **Visual strobe**: Connect a Red LED on GP11 to flash in sync with the buzzer siren.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly on boot | Calibration phase active | Wait 30 seconds after powering on for the PIR sensor to calibrate. |
| Reset button does not disarm | Button pin mismatch | Verify the reset button is wired to GP13 as configured in the code. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [74 - Pico Motion Security PIR Alarm](74-pico-motion-security-pir-alarm.md)
- [81 - Pico Tilt Alarm LCD](81-pico-tilt-alarm-lcd.md)
