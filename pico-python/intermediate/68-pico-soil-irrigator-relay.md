# 68 - Pico Soil Irrigator Relay

Build an automated plant watering station that monitors soil moisture and opens an irrigation solenoid valve (relay) when the soil is dry.

## Goal
Learn how to monitor analog soil moisture sensors, implement threshold control logic, and actuate high-current relays in MicroPython.

## What You Will Build
An automatic plant waterer:
- **Soil Moisture Sensor (GP26)**: Measures moisture level.
- **Relay Module (GP15)**: Switches power ON to an irrigation solenoid water valve when the soil moisture falls below the dry threshold, and OFF once the soil is watered.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Soil Moisture Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| Relay Module | `relay` | Yes | Yes (controls solenoid valve) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Soil Sensor | VCC (+) | 3.3V (3V3) | Red | Power line |
| Soil Sensor | GND (−) | GND | Black | Ground reference |
| Soil Sensor | OUT (Analog) | GP26 | Yellow | Analog input signal |
| Relay Module | IN | GP15 | Orange | Relay control signal (HIGH on activate) |
| Relay Module | VCC / GND | 5V (VBUS) / GND | Red / Black | Driver power supply |

> **Wiring tip:** Connect the soil moisture sensor's output to GP26, and the relay control input pin to GP15. Connect the relay VCC to 5V VBUS.

## Code
```python
from machine import Pin, ADC
import utime

soil_sensor = ADC(26) # GP26 = ADC channel 0
valve_relay = Pin(15, Pin.OUT)

# Calibration thresholds (raw ADC scale 0 - 65535, high = dry)
DRY_LIMIT  = 38000 # Turn valve ON if reading > 38000 (Soil is dry)
WET_LIMIT  = 22000 # Turn valve OFF if reading < 22000 (Soil is watered)

valve_relay.value(0) # Start closed (OFF)
watering_active = False

print("Soil irrigation controller armed.")

while True:
    raw = soil_sensor.read_u16()
    print("Soil Moisture Reading:", raw, "| Irrigation:", "ACTIVE" if watering_active else "OFF")
    
    # Hysteresis control logic
    if raw > DRY_LIMIT and not watering_active:
        watering_active = True
        valve_relay.value(1) # Open water valve
        print(">> Soil is dry! Starting watering...")
    elif raw < WET_LIMIT and watering_active:
        watering_active = False
        valve_relay.value(0) # Close water valve
        print(">> Soil watered. Stopping watering.")
        
    utime.sleep(1) # 1-second update cycle
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Soil Moisture Sensor** (represented by potentiometer), and **Relay** onto the canvas.
2. Connect Sensor to **GP26** and Relay to **GP15**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the soil sensor to maximum (dry) to see the relay activate, then slide it down below 22000 to see it turn OFF.

## Expected Output
```
Soil irrigation controller armed.
Soil Moisture Reading: 45000 | Irrigation: OFF
>> Soil is dry! Starting watering...
Soil Moisture Reading: 18000 | Irrigation: ACTIVE
>> Soil watered. Stopping watering.
```

## Expected Canvas Behavior
- The relay component clicks active and closes the switch contacts when the soil sensor slider is moved to a high value (representing dry soil).

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `raw > DRY_LIMIT` | Checks if the soil moisture has dropped (resulting in a higher resistance/voltage reading) below the dry limit. |
| `valve_relay.value(1)` | Drives GP15 HIGH to open the water solenoid valve. |
| `raw < WET_LIMIT` | Checks if the soil is wet enough to turn the water valve OFF. |

## Hardware & Safety Concept: Auto-Shutoff Timeouts
If a soil moisture sensor becomes disconnected or fails, it will report dry soil indefinitely. In an automated system, this would cause the water valve to stay open, flooding the area. To prevent flooding, smart irrigation systems implement a **maximum runtime safety limit** (e.g. closing the valve if watering runs for longer than 60 seconds) and trigger a sensor failure alarm.

## Try This! (Challenges)
1. **Critical Water Timeout**: Modify the code to count the number of seconds the valve has been open. If it exceeds 10 seconds, close the valve and lock out the system until rebooted.
2. **Audio Indicator**: Connect a buzzer on GP14 to beep while the valve is open.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Valve does not open when soil is dry | Sensor wire disconnected | Check the connection between the sensor OUT pin and GP26, and verify raw ADC readings in the terminal. |
| Valve stays open constantly | Hysteresis thresholds incorrect | Adjust the `DRY_LIMIT` and `WET_LIMIT` constants in your code to match your specific sensor's dry and wet readings. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [42 - Pico Soil Moisture Serial](42-pico-soil-moisture-serial.md)
- [62 - Pico Relay Button](62-pico-relay-button.md)
- [67 - Pico Water Level Pump Relay](67-pico-water-level-pump-relay.md)
