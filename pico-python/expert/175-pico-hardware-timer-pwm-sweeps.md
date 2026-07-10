# 175 - Pico Hardware Timer PWM Tone Sweeps

Build an audio siren and frequency generator that uses a hardware timer to sweep a PWM passive buzzer across different sound profiles in the background.

## Goal
Learn how to use hardware timers to continuously modify PWM output parameters, generate dynamic pitch sweeps, and build a non-blocking audio engine in MicroPython.

## What You Will Build
A background warning siren generator:
- **Passive Buzzer (GP15)**: Generates audio tones using variable PWM frequency.
- **Button A (GP13)**: Starts or stops the siren.
- **Button B (GP14)**: Cycles between three sound profiles (Police Siren, Laser Sweep, Yelp Alarm).
- **16x2 I2C LCD (GP4, GP5)**: Displays the active audio profile and current output frequency.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Passive Buzzer Module | `buzzer` | Yes | Yes |
| Tactile Push Button × 2 | `button` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Passive Buzzer | Signal (PWM) | GP15 | Orange | Audio tone output pin |
| Passive Buzzer | GND | GND | Black | Ground return |
| Button A | Terminal 1 | GP13 | White | Start/stop toggle |
| Button A | Terminal 2 | GND | Black | Ground return |
| Button B | Terminal 1 | GP14 | Yellow | Sound profile selector |
| Button B | Terminal 2 | GND | Black | Ground return |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** A passive buzzer requires a PWM signal to vibrate at specific frequencies. Connect its signal pin to GP15. Buttons connect to GP13 and GP14.

## Code
```python
from machine import Pin, PWM, I2C, Timer
import utime
from machine_lcd import I2cLcd

buzzer = PWM(Pin(15))
buzzer.duty_u16(0)  # Start silent

btn_start = Pin(13, Pin.IN, Pin.PULL_UP)
btn_mode = Pin(14, Pin.IN, Pin.PULL_UP)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Sound parameters
active = False
sound_mode = 0  # 0=Yelp, 1=Laser, 2=Siren
mode_names = ["Yelp Alarm", "Laser Sweep", "Siren Alert"]

# Sweep variables
freq = 500
direction = 1
sweep_timer = Timer()

def play_tone(f):
    if f < 50: f = 50
    buzzer.freq(f)
    buzzer.duty_u16(32768)  # 50% duty cycle

def mute():
    buzzer.duty_u16(0)

def sweep_callback(timer):
    """Callback function triggered every 15 ms to calculate and update frequency."""
    global freq, direction
    if not active:
        mute()
        return

    if sound_mode == 0:  # Yelp: Fast rising sweep
        freq += 15
        if freq > 1500:
            freq = 500
        play_tone(freq)
        
    elif sound_mode == 1:  # Laser: Falling chirp
        freq -= 25
        if freq < 400:
            freq = 2000
        play_tone(freq)
        
    elif sound_mode == 2:  # Police Siren: Alternating high/low
        # Oscillates between 600 Hz and 1200 Hz
        if direction == 1:
            freq += 8
            if freq >= 1200:
                direction = -1
        else:
            freq -= 8
            if freq <= 600:
                direction = 1
        play_tone(freq)

lcd.clear()
lcd.putstr("Siren Engine\nPress Start...")
utime.sleep(1.0)

# Initialize background sweep timer at 15 ms interval
sweep_timer.init(period=15, mode=Timer.PERIODIC, callback=sweep_callback)

print("Siren generator ready.")

last_btn = 0

while True:
    now = utime.ticks_ms()
    
    # Toggle Active State (Button A)
    if btn_start.value() == 0 and utime.ticks_diff(now, last_btn) > 400:
        active = not active
        last_btn = now
        print("Siren Active State:", active)
        if not active:
            mute()
            
    # Cycle Modes (Button B)
    if btn_mode.value() == 0 and utime.ticks_diff(now, last_btn) > 400:
        sound_mode = (sound_mode + 1) % 3
        last_btn = now
        # Reset base frequency for mode
        if sound_mode == 0: freq = 500
        elif sound_mode == 1: freq = 2000
        else: freq = 600
        print("Siren Mode:", mode_names[sound_mode])
        
    # Update LCD
    lcd.clear()
    if active:
        lcd.putstr("M:{}\n".format(mode_names[sound_mode]))
        lcd.putstr("Freq: {:4d} Hz".format(freq))
    else:
        lcd.putstr("M:{}\n".format(mode_names[sound_mode]))
        lcd.putstr("Status: MUTED ")
        
    utime.sleep_ms(200)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Passive Buzzer**, **two Buttons**, and **I2C LCD** onto the canvas.
2. Connect Buzzer to **GP15**, Buttons to **GP13/GP14**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Press Button A to start the alarm, and press Button B to cycle through Yelp, Laser, and Siren effects.

## Expected Output
Terminal:
```
Siren generator ready.
Siren Active State: True
Siren Mode: Laser Sweep
Siren Mode: Siren Alert
```

## Expected Canvas Behavior
* Pressing Button A starts the passive buzzer tone generator. The frequency value on the LCD changes continuously.
* Pressing Button B alters the audio pitch sweep pattern (e.g. rising Yelp vs. falling Laser).
* Pressing Button A again stops the buzzer immediately.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `buzzer.duty_u16(32768)` | Sets the PWM duty cycle to 50% (half of 65536), which generates the maximum volume square wave. |
| `period=15` | Configures the hardware timer to trigger its callback every 15 milliseconds. |

## Hardware & Safety Concept: Non-Blocking Audio Warnings
Legacy alert systems use blocking loops: `tone(1000); delay(500); tone(500); delay(500)`. During these delays, the system cannot respond to emergency input signals or safety cut-off triggers. Using hardware timers to increment/decrement frequency registers ensures that the audio plays smoothly in the background, keeping the main loop completely responsive to sensor inputs and safety switches.

## Try This! (Challenges)
1. **Pitch Knob**: Connect a potentiometer on GP26 and use it to offset the base frequency of the sweep, shifting the pitch scale higher or lower.
2. **Visual Warning Flash**: Add a Red LED on GP16 and toggle it ON/OFF in sync with the sweep peaks to create a strobe effect.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer clicks but plays no clear tone | Frequency set too low | Ensure `freq` never drops below 50 Hz. Piezo elements cannot vibrate cleanly at extremely low frequencies. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [06 - Active Buzzer Alarm Pattern](../../beginner/06-active-buzzer-alarm-pattern.md)
- [70 - Active Buzzer Chime scale generator](../intermediate/70-active-buzzer-chime-scale-generator.md)
