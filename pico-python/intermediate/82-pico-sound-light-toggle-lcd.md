# 82 - Pico Sound Light Toggle LCD

Build a clap-activated switch console that toggles a light (LED/relay) state when a loud noise is detected, and displays active switch states on an I2C LCD.

## Goal
Learn how to read digital inputs from sound sensor comparators, implement state toggling, handle noise debouncing, and update status text on an I2C character LCD in MicroPython.

## What You Will Build
A clap switch console with display:
- **Sound Sensor (GP14)**: Detects loud sounds (active LOW digital signal).
- **LED/Relay (GP15)**: Toggles its state ON and OFF on alternate claps.
- **I2C 16x2 LCD (GP4, GP5)**: Displays the active state of the light ("LIGHT STATE: ON" or "LIGHT STATE: OFF").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Sound Sensor Module | `button` | Yes (represented by push button component) | Yes |
| LED (any colour) | `led` | Yes | Yes (simulates relay/light load) |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Sound Sensor | VCC | 3.3V (3V3) or 5V | Red | Power line |
| Sound Sensor | GND | GND | Black | Ground reference |
| Sound Sensor | DO (Digital Out) | GP14 | Blue | Digital sound trigger (LOW on sound) |
| LED | Anode (+, longer leg) | GP15 | Orange | Relay switch output pin |
| LED | Cathode (−, shorter leg) | GND | Black | Shared return path to ground |
| 330 Ω Resistor | Either leg | In series with LED Anode | — | Limits LED current |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the sound sensor's digital output pin to GP14 (with internal pull-up). Connect the LED anode to GP15 through the current-limiting resistor. Connect the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime
from machine_lcd import I2cLcd

sound_sensor = Pin(14, Pin.IN, Pin.PULL_UP)
relay_light  = Pin(15, Pin.OUT)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

light_state = False
relay_light.value(0) # Start OFF

prev_val = 1
sound_sensor.value()

# Initial LCD update
lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("Clap Switch Node")
lcd.move_to(0, 1)
lcd.putstr("Light State: OFF")

print("Clap switch active. Make some noise!")

while True:
    curr_val = sound_sensor.value()
    
    # Detect falling edge: HIGH -> LOW (sound trigger)
    if prev_val == 1 and curr_val == 0:
        light_state = not light_state # Toggle light state
        relay_light.value(1 if light_state else 0)
        
        # Update LCD Display
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("Clap Detected!")
        lcd.move_to(0, 1)
        lcd.putstr("Light: " + ("ON " if light_state else "OFF"))
        
        print("Clap detected! Light:", "ON" if light_state else "OFF")
        
        # Debounce delay: ignore sound reflections for 400 ms
        utime.sleep_ms(400)
        
    prev_val = curr_val
    utime.sleep_ms(10) # Fast polling for noise peaks
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Push Button** (representing sound sensor), **LED**, and **I2C LCD** onto the canvas.
2. Connect Button to **GP14**, LED to **GP15**, and LCD to **GP4/GP5**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the button component on the canvas to simulate a clap and observe the LED and LCD display update.

## Expected Output
```
Clap switch active. Make some noise!
Clap detected! Light: ON
Clap detected! Light: OFF
```
(On screen: "Light: ON" or "Light: OFF" on line 2, updating with claps.)

## Expected Canvas Behavior
- The LED component on the canvas toggles and the LCD screen updates to show the light state on alternate clicks of the sound button.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `sound_sensor.value() == 0` | Checks if the sound sensor board comparator has pulled GP14 LOW, indicating a volume peak. |
| `utime.sleep_ms(400)` | Debounce window that prevents sound reflections from causing rapid toggle states. |

## Hardware & Safety Concept: Sound Debounce Windows
When a hand claps, it produces a primary sound wave followed by multiple echoes reflecting off walls. Without a software **debounce lock out window** (typically 300 to 500 ms), the microcontroller would register these reflections as separate claps, causing the light to toggle ON and OFF rapidly and end up in a random state.

## Try This! (Challenges)
1. **Double Clap Switch**: Modify the code to require two distinct claps within 1 second to toggle the light, reducing false triggers from background noise.
2. **Audio Feedback**: Connect a buzzer on GP13 to sound a short beep when the state changes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Light toggles constantly | Sensitivity set too high | On real hardware, rotate the onboard potentiometer screw on the microphone board to adjust the noise threshold. |
| LCD does not update on clap | Polling speed mismatch | Ensure `sleep_ms` is set to a fast polling speed (10 ms) to capture short acoustic pulses. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [71 - Pico Sound Light Toggle](71-pico-sound-light-toggle.md)
