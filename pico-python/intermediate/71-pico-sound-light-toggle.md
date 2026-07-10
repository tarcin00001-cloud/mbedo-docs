# 71 - Pico Sound Light Toggle

Build a clap-activated switch that toggles the state of an LED (or power relay) ON and OFF when a loud noise is detected.

## Goal
Learn how to read digital inputs from sound sensor comparators, implement state toggling, and handle noise debouncing in MicroPython.

## What You Will Build
A clap switch controller:
- **Sound Sensor (GP14)**: Detects loud sounds (active LOW digital signal from comparator).
- **LED/Relay (GP15)**: Toggles its state between ON and OFF each time a clap/loud noise is registered.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Sound Sensor Module | `button` | Yes (represented by push button component) | Yes (microphone with digital out) |
| LED (any colour) | `led` | Yes | Yes (simulates relay/light load) |
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

> **Wiring tip:** Connect the sound sensor's digital output pin to GP14 (with internal pull-up). Connect the LED anode to GP15 through the current-limiting resistor. All grounds are shared.

## Code
```python
from machine import Pin
import utime

sound_sensor = Pin(14, Pin.IN, Pin.PULL_UP)
relay_light  = Pin(15, Pin.OUT)

light_state = False
relay_light.value(0) # Start OFF

prev_val = 1
sound_sensor.value()

print("Clap switch active. Make some noise!")

while True:
    curr_val = sound_sensor.value()
    
    # Detect falling edge: HIGH -> LOW (sound trigger)
    if prev_val == 1 and curr_val == 0:
        light_state = not light_state # Toggle light state
        relay_light.value(1 if light_state else 0)
        
        print("Clap detected! Light:", "ON" if light_state else "OFF")
        
        # Debounce delay: ignore sound reflections for 300 ms
        utime.sleep_ms(300)
        
    prev_val = curr_val
    utime.sleep_ms(10) # Fast polling for noise peaks
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Push Button** (representing sound sensor), and **LED** onto the canvas.
2. Connect Button to **GP14** and LED to **GP15**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the button component on the canvas to simulate a clap and observe the LED toggling.

## Expected Output
```
Clap switch active. Make some noise!
Clap detected! Light: ON
Clap detected! Light: OFF
```

## Expected Canvas Behavior
- The LED component on the canvas toggles between glowing and off on alternate clicks of the sound button.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `sound_sensor.value() == 0` | Checks if the sound sensor board comparator has pulled GP14 LOW, indicating a volume peak. |
| `utime.sleep_ms(300)` | Critical debounce window that prevents the echo/reverberation of a single clap from triggering multiple toggles. |

## Hardware & Safety Concept: Sound Debounce Windows
When a hand claps, it produces a primary sound wave followed by multiple echoes reflecting off walls. Without a software **debounce lock out window** (typically 300 to 500 ms), the microcontroller would register these reflections as separate claps, causing the light to toggle ON and OFF rapidly and end up in a random state.

## Try This! (Challenges)
1. **Double Clap Switch**: Modify the code to require two distinct claps within 1 second to toggle the light, reducing false triggers from background noise.
2. **Audio Feedback**: Connect a buzzer on GP13 to sound a short beep when the state changes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Light toggles constantly | Sensitivity set too high | On real hardware, rotate the onboard potentiometer screw on the microphone board to adjust the noise threshold. |
| Light does not toggle on clap | Threshold too low | Adjust the sensor board sensitivity screw until the onboard indicator LED flashes on claps. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [03 - Pico Button LED](03-pico-button-led.md)
- [45 - Pico Sound Sensor Serial](45-pico-sound-sensor-serial.md)
- [62 - Pico Relay Button](62-pico-relay-button.md)
