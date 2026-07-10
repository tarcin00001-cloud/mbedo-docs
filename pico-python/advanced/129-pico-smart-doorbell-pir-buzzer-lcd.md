# 129 - Pico Smart Doorbell PIR Buzzer LCD

Build a smart doorbell system that detects a visitor using a PIR motion sensor, plays a two-tone chime on a passive buzzer, increments a visitor counter, and displays the last detection time and visit count on an I2C LCD.

## Goal
Learn how to use a PIR sensor as a visitor detection trigger, produce a two-tone doorbell chime using PWM on a passive buzzer, maintain a running visitor counter, and display event history on an I2C character LCD in MicroPython.

## What You Will Build
A smart doorbell:
- **PIR Sensor (GP14)**: Detects visitors approaching the door.
- **Passive Buzzer (GP15)**: Plays a "Ding-Dong" two-tone chime on detection.
- **LED (GP13)**: Flashes briefly on each detection event.
- **I2C 16x2 LCD (GP4, GP5)**: Displays visitor count and "Visitor!" alert.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| PIR Motion Sensor | `button` | Yes (digital output) | Yes |
| Passive Buzzer | `buzzer` | Yes (passive type required) | Yes |
| LED | `led` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| PIR Sensor | VCC | 5V (VBUS) | Red | PIR power supply |
| PIR Sensor | GND | GND | Black | Ground reference |
| PIR Sensor | OUT | GP14 | Blue | Motion output (HIGH on detection) |
| Passive Buzzer | Signal (+) | GP15 | Grey | Chime PWM output |
| Passive Buzzer | GND (−) | GND | Black | Ground return |
| LED | Anode (+, via 330 Ω) | GP13 | Orange | Detection flash indicator |
| LED | Cathode (−) | GND | Black | Ground return |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the PIR output to GP14. Connect the passive buzzer to GP15 (PWM-capable). Connect the LED to GP13 through a 330 Ω resistor. Connect the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, PWM, I2C
import utime
from machine_lcd import I2cLcd

pir    = Pin(14, Pin.IN, Pin.PULL_DOWN)
buzzer = PWM(Pin(15))
led    = Pin(13, Pin.OUT)

buzzer.duty_u16(0)

# I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

visit_count = 0
last_visit  = "No visitors yet"

LOCKOUT_MS = 5000  # Minimum ms between detections (prevent re-trigger)
last_detect = utime.ticks_ms() - LOCKOUT_MS  # Allow immediate first detection

def play_chime():
    """Play a Ding-Dong two-tone chime."""
    # "Ding" — higher note
    buzzer.freq(1047)  # C6
    buzzer.duty_u16(32768)
    utime.sleep_ms(300)
    buzzer.duty_u16(0)
    utime.sleep_ms(80)

    # "Dong" — lower note
    buzzer.freq(784)   # G5
    buzzer.duty_u16(32768)
    utime.sleep_ms(400)
    buzzer.duty_u16(0)

def flash_led():
    led.value(1)
    utime.sleep_ms(200)
    led.value(0)

lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("Smart Doorbell")
lcd.move_to(0, 1)
lcd.putstr("Ready...")
utime.sleep(2)

print("Smart doorbell active. Waiting for visitors...")

while True:
    now = utime.ticks_ms()

    if pir.value() == 1:
        # Lockout check
        if utime.ticks_diff(now, last_detect) >= LOCKOUT_MS:
            last_detect  = now
            visit_count += 1

            # Format visit number as display-friendly string
            last_visit = "Visit #{}".format(visit_count)

            print(">> Visitor detected! Total visits:", visit_count)

            # Flash and chime
            flash_led()
            play_chime()

            # Update LCD
            lcd.clear()
            lcd.move_to(0, 0)
            lcd.putstr(">> VISITOR! <<  ")
            lcd.move_to(0, 1)
            lcd.putstr("Count: {:4d}     ".format(visit_count))
            utime.sleep(2)
        else:
            # Still in lockout period
            pass
    else:
        # Standby display
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("Doorbell Ready  ")
        lcd.move_to(0, 1)
        lcd.putstr("Visits: {:4d}    ".format(visit_count))

    utime.sleep_ms(200)
```

## What to Click in MbedO
1. Drag **Pico**, **PIR Button**, **Passive Buzzer**, **LED**, and **I2C LCD** onto the canvas.
2. Connect PIR to **GP14**, Buzzer to **GP15**, LED to **GP13**, and LCD to **GP4/GP5**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the PIR button to simulate a visitor — observe the LED flash, chime play, and LCD update with the visitor count.

## Expected Output
```
Smart doorbell active. Waiting for visitors...
>> Visitor detected! Total visits: 1
>> Visitor detected! Total visits: 2
```
(On screen: ">> VISITOR! <<" flashes for 2 seconds, then returns to "Doorbell Ready | Visits: 2".)

## Expected Canvas Behavior
- Clicking the PIR button triggers the LED to flash, the buzzer to play the Ding-Dong tone sequence, and the LCD to show the visitor alert and incrementing count.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `utime.ticks_diff(now, last_detect) >= LOCKOUT_MS` | Prevents multiple counts from a single visitor by enforcing a 5-second gap between detections. |
| `buzzer.freq(1047)` / `.freq(784)` | Sets the PWM frequency to C6 (Ding) and G5 (Dong) to produce the two-tone chime. |

## Hardware & Safety Concept: PIR Re-Trigger Modes
PIR sensors have two output modes selectable via an onboard jumper: **single-trigger (L mode)** — outputs one pulse per motion event, and **re-trigger (H mode)** — holds HIGH as long as motion continues. For a doorbell, single-trigger (L mode) combined with a software lockout gives the most reliable visitor count.

## Try This! (Challenges)
1. **Custom Melody**: Replace the two-tone chime with a full "Westminster Chimes" four-note sequence.
2. **Night Silence Mode**: Connect an LDR on GP26 and suppress the buzzer chime after midnight (simulated by the LDR reading) while still flashing the LED.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer clicks but no tone | Active buzzer connected | Use a **passive** buzzer — active buzzers have internal oscillators and cannot produce variable tones. |
| Multiple counts per visitor | Lockout too short | Increase `LOCKOUT_MS` to 8000 (8 seconds) to account for longer PIR output pulses. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [101 - Pico Doorbell Double Chime](../intermediate/101-pico-doorbell-double-chime.md)
- [85 - Pico Motion Security PIR LCD](../intermediate/85-pico-motion-security-pir-lcd.md)
