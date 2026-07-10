# 105 - Pico Soil Drought Alarm

Build a soil moisture drought alarm that flashes a warning LED and sounds a pulsing warning buzzer when soil dryness levels exceed danger thresholds.

## Goal
Learn how to read analog inputs from soil moisture sensors, implement threshold alarm logic, and control warning LEDs and active buzzers in MicroPython.

## What You Will Build
A plant drought warning system:
- **Soil Moisture Sensor (GP26)**: Monitors soil moisture levels.
- **LED (GP14)**: Flashes during alarm states.
- **Active Buzzer (GP15)**: Sounds a pulsing warning siren when triggered.
- **Reset Button (GP13)**: Silences the alarm once watered.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Soil Moisture Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| LED (any colour) | `led` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Tactile Push Button | `button` | Yes (reset switch) | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Soil Sensor | VCC (+) | 3.3V (3V3) | Red | Power line |
| Soil Sensor | GND (−) | GND | Black | Ground reference |
| Soil Sensor | OUT (Analog) | GP26 | Yellow | Analog input signal |
| Reset Button | Terminal 1 | GP13 | White | Reset input (reads LOW on press) |
| Reset Button | Terminal 2 | GND | Black | Shorts GP13 to GND |
| LED | Anode (+, longer leg) | GP14 | Orange | Warning indicator output pin |
| LED | Cathode (−, shorter leg) | GND | Black | Shared return path to ground |
| 330 Ω Resistor | Either leg | In series with LED Anode | — | Limits LED current |
| Active Buzzer | VCC (+) | GP15 | Red | Alarm sound output pin |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the soil moisture sensor output to GP26. Connect the reset button to GP13, LED anode to GP14, and active buzzer VCC to GP15. All grounds are shared. Note: soil sensors output HIGH when dry (low conductivity) and LOW when wet.

## Code
```python
from machine import Pin, ADC
import utime

soil_sensor = ADC(26) # GP26 = ADC channel 0
btn_reset   = Pin(13, Pin.IN, Pin.PULL_UP)
warning_led = Pin(14, Pin.OUT)
buzzer      = Pin(15, Pin.OUT)

# Threshold: alarm triggers if soil reading > this (high = dry, raw scale 0 - 65535)
DROUGHT_LIMIT = 40000

alarm_state = False
warning_led.value(0)
buzzer.value(0)

print("Soil drought alarm active.")

while True:
    raw = soil_sensor.read_u16()
    reset_pressed = (btn_reset.value() == 0)
    
    # High reading = dry soil (low conductivity)
    if raw > DROUGHT_LIMIT and not alarm_state:
        alarm_state = True
        print(">> WARNING: Soil drought detected! Alarm active. Level:", raw)
        
    # Check for disarm (Reset)
    if reset_pressed and alarm_state:
        alarm_state = False
        warning_led.value(0)
        buzzer.value(0)
        print(">> Drought alarm reset.")
        utime.sleep(1) # Lockout delay
        
    # Pulsing warning siren
    if alarm_state:
        warning_led.value(1)
        buzzer.value(1)
        utime.sleep_ms(200)
        warning_led.value(0)
        buzzer.value(0)
        utime.sleep_ms(200)
    else:
        utime.sleep(1) # Normal polling speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Soil Moisture Sensor** (represented by potentiometer), **Push Button** (reset switch), **LED**, and **Active Buzzer** onto the canvas.
2. Connect Soil Sensor wiper to **GP26**.
3. Connect Reset Button to **GP13**, LED to **GP14**, and Buzzer to **GP15**. Connect all grounds.
4. Paste the code, select **MicroPython** mode, and click **Run**.
5. Slide the soil sensor to maximum (dry) to trigger the alarm. Click Reset to disarm.

## Expected Output
```
Soil drought alarm active.
>> WARNING: Soil drought detected! Alarm active. Level: 43500
>> Drought alarm reset.
```

## Expected Canvas Behavior
- The LED and buzzer components on the canvas pulse ON and OFF rapidly when the soil moisture sensor slider is moved above 40000 (simulating dry soil).
- Clicking the Reset Button silences the alarm.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `raw > DROUGHT_LIMIT` | Checks if the soil reading is above the dry threshold (high raw value = dry soil = low conductivity). |
| `alarm_state = True` | Latches the alarm state so the siren continues even if the sensor value drops. |

## Hardware & Safety Concept: Sensor Fail-Safe Protection
Soil moisture sensors placed in active dirt are exposed to moisture, fertilizer acids, and mechanical stress. Sensor failure (e.g. broken signal wire) typically causes the analog reading to drift high (dry soil state). To prevent the system from alarming indefinitely on sensor failure, smart irrigation systems implement a **maximum alarm runtime cutoff** after which the system enters a maintenance alert mode.

## Try This! (Challenges)
1. **Auto-Watering**: Connect a relay on GP12 that automatically opens a water valve when the drought alarm triggers.
2. **LCD Panel**: Add an I2C LCD on GP4/GP5 to display the moisture percentage alongside the alarm state.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly | Threshold set too low | Print soil values to the terminal in normal conditions, then adjust `DROUGHT_LIMIT` accordingly. |
| Alarm does not disarm | Reset button pin mismatch | Verify the reset button is wired to GP13 as configured in the code. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [42 - Pico Soil Moisture Serial](42-pico-soil-moisture-serial.md)
- [68 - Pico Soil Irrigator Relay](68-pico-soil-irrigator-relay.md)
- [79 - Pico Soil Irrigator LCD](79-pico-soil-irrigator-lcd.md)
