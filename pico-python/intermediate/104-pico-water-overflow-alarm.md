# 104 - Pico Water Overflow Alarm

Build a water level safety overflow alarm that flashes a warning LED and sounds a pulsing warning buzzer when liquid levels exceed flood safety limits.

## Goal
Learn how to read analog inputs from liquid level sensors, implement threshold alarm logic, and control warning LEDs and active buzzers in MicroPython.

## What You Will Build
A flood warning system:
- **Water Level Sensor (GP26)**: Monitors water depth.
- **LED (GP14)**: Flashes during alarm states.
- **Active Buzzer (GP15)**: Sounds a pulsing warning siren when triggered.
- **Reset Button (GP13)**: Silences the alarm once the flood clears.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Water Level Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| LED (any colour) | `led` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Tactile Push Button | `button` | Yes (reset switch) | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Water Level Sensor | VCC (+) | 3.3V (3V3) | Red | Power line |
| Water Level Sensor | GND (−) | GND | Black | Ground reference |
| Water Level Sensor | OUT (Analog) | GP26 | Yellow | Analog input signal |
| Reset Button | Terminal 1 | GP13 | White | Reset input (reads LOW on press) |
| Reset Button | Terminal 2 | GND | Black | Shorts GP13 to GND |
| LED | Anode (+, longer leg) | GP14 | Orange | Warning indicator output pin |
| LED | Cathode (−, shorter leg) | GND | Black | Shared return path to ground |
| 330 Ω Resistor | Either leg | In series with LED Anode | — | Limits LED current |
| Active Buzzer | VCC (+) | GP15 | Red | Alarm sound output pin |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the water level sensor output to GP26. Connect the reset button to GP13, LED anode to GP14, and active buzzer VCC to GP15. All grounds are shared.

## Code
```python
from machine import Pin, ADC
import utime

water_sensor = ADC(26) # GP26 = ADC channel 0
btn_reset    = Pin(13, Pin.IN, Pin.PULL_UP)
warning_led  = Pin(14, Pin.OUT)
buzzer       = Pin(15, Pin.OUT)

# Threshold: alarm triggers if water level exceeds this (raw scale 0 - 65535)
FLOOD_LIMIT = 35000

alarm_state = False
warning_led.value(0)
buzzer.value(0)

print("Water overflow safety alarm active.")

while True:
    raw = water_sensor.read_u16()
    reset_pressed = (btn_reset.value() == 0)
    
    # Check for trigger (Set)
    if raw > FLOOD_LIMIT and not alarm_state:
        alarm_state = True
        print(">> WARNING: Water overflow detected! Alarm active. Level:", raw)
        
    # Check for disarm (Reset)
    if reset_pressed and alarm_state:
        alarm_state = False
        warning_led.value(0)
        buzzer.value(0)
        print(">> Overflow alarm reset.")
        utime.sleep(1) # Lockout delay
        
    # Pulsing warning siren
    if alarm_state:
        warning_led.value(1)
        buzzer.value(1)
        utime.sleep_ms(150)
        warning_led.value(0)
        buzzer.value(0)
        utime.sleep_ms(150)
    else:
        utime.sleep(1) # Normal polling speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Water Level Sensor** (represented by potentiometer), **Push Button** (reset switch), **LED**, and **Active Buzzer** onto the canvas.
2. Connect Water Level Sensor wiper to **GP26**.
3. Connect Reset Button to **GP13**, LED to **GP14**, and Buzzer to **GP15**. Connect all grounds.
4. Paste the code, select **MicroPython** mode, and click **Run**.
5. Slide the water level sensor past 35000 to trigger the alarm. Click Reset to disarm.

## Expected Output
```
Water overflow safety alarm active.
>> WARNING: Water overflow detected! Alarm active. Level: 38200
>> Overflow alarm reset.
```

## Expected Canvas Behavior
- The LED and buzzer components on the canvas pulse ON and OFF rapidly when the water level sensor slider is moved past 35000.
- Clicking the Reset Button silences the alarm and turns the LED OFF.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `raw > FLOOD_LIMIT` | Checks if the measured water level is above the safety threshold, indicating an overflow. |
| `alarm_state = True` | Latches the alarm state to `True` so the alarm continues to sound even if the water drains. |

## Hardware & Safety Concept: Alarm Latching and Reset Loops
Safety standards require **alarm latching** for critical parameters. Once a threshold is crossed, the alarm must continue to sound even if the parameter returns to safe limits (e.g. the water level falls). This ensures that temporary hazard events are noticed by operators, and the system can only be silenced by an authorized manual reset.

## Try This! (Challenges)
1. **Visual strobe**: Connect a Red LED on GP12 and flash it in sync with the buzzer siren.
2. **Sump Pump Relay**: Connect an LED/Relay on GP12 that turns ON (simulating a sump pump) when the overflow alarm is active.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly | Threshold set too low | Print water level values to the terminal to find the dry level, then adjust `FLOOD_LIMIT` in your code. |
| Alarm does not disarm | Reset button pin mismatch | Verify the reset button is wired to GP13 as configured in the code. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [41 - Pico Water Level Serial](41-pico-water-level-serial.md)
- [67 - Pico Water Level Pump Relay](67-pico-water-level-pump-relay.md)
- [78 - Pico Water Level Pump LCD](78-pico-water-level-pump-lcd.md)
