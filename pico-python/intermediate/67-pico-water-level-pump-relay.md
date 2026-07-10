# 67 - Pico Water Level Pump Relay

Build an automated sump pump control system that switches ON a water pump relay when water level rises too high, protecting basins from overflow.

## Goal
Learn how to monitor analog liquid level sensors, implement threshold switch logic, and control high-power relays in MicroPython.

## What You Will Build
An automatic sump pump controller:
- **Water Level Sensor (GP26)**: Measures water depth.
- **Relay Module (GP15)**: Switches power ON to a simulated water pump when water exceeds the high threshold, and OFF when it drops below the low threshold (preventing pump dry-running).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Water Level Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes (resistive sensor) |
| Relay Module | `relay` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Water Level Sensor | VCC (+) | 3.3V (3V3) | Red | Power line |
| Water Level Sensor | GND (−) | GND | Black | Ground reference |
| Water Level Sensor | OUT (Analog) | GP26 | Yellow | Analog input signal |
| Relay Module | IN | GP15 | Orange | Relay control signal (HIGH on activate) |
| Relay Module | VCC / GND | 5V (VBUS) / GND | Red / Black | Driver power supply |

> **Wiring tip:** Connect the water level sensor's output to GP26, and the relay control input pin to GP15. Connect the relay VCC to 5V VBUS.

## Code
```python
from machine import Pin, ADC
import utime

water_sensor = ADC(26) # GP26 = ADC channel 0
pump_relay   = Pin(15, Pin.OUT)

# Calibration limits (raw ADC range 0 - 65535)
PUMP_ON_LIMIT  = 35000 # Turn pump ON if water level > 35000
PUMP_OFF_LIMIT = 15000 # Turn pump OFF if water level < 15000

pump_relay.value(0) # Start OFF
pump_state = False

print("Sump pump controller armed.")

while True:
    raw = water_sensor.read_u16()
    print("Water Level Reading:", raw, "| Pump State:", "ON" if pump_state else "OFF")
    
    # Dual-threshold control logic
    if raw > PUMP_ON_LIMIT and not pump_state:
        pump_state = True
        pump_relay.value(1) # Turn pump ON
        print(">> Basins flooding! Pump ACTIVATED.")
    elif raw < PUMP_OFF_LIMIT and pump_state:
        pump_state = False
        pump_relay.value(0) # Turn pump OFF
        print(">> Basins drained. Pump DEACTIVATED.")
        
    utime.sleep(1) # 1-second update cycle
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Water Level Sensor** (represented by potentiometer), and **Relay** onto the canvas.
2. Connect Sensor to **GP26** and Relay to **GP15**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the water sensor to maximum (flood) to see the relay activate, then slide it down below 15000 to see it turn OFF.

## Expected Output
```
Sump pump controller armed.
Water Level Reading: 8000 | Pump State: OFF
Water Level Reading: 42000 | Pump State: OFF
>> Basins flooding! Pump ACTIVATED.
Water Level Reading: 12000 | Pump State: ON
>> Basins drained. Pump DEACTIVATED.
```

## Expected Canvas Behavior
- The relay component clicks active and closes the switch contacts when the water sensor slider is moved to a high value.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `raw > PUMP_ON_LIMIT` | Checks if the water depth is high enough to warrant starting the pump. |
| `pump_relay.value(1)` | Drives GP15 HIGH to activate the relay, starting the water pump. |
| `raw < PUMP_OFF_LIMIT` | Checks if the water has dropped below the safety limit to turn the pump OFF. |

## Hardware & Safety Concept: Dry-Run Pump Protection
Water pumps rely on the fluid they are moving to cool and lubricate their internal seals. Running a pump when the sump basin is completely empty (dry running) can quickly ruin the motor. Sump pump controllers must use separate ON and OFF thresholds (hysteresis) to ensure the pump runs only when there is water, and shuts down immediately if water levels drop below the safety limit.

## Try This! (Challenges)
1. **High Flood Warning**: Connect a buzzer on GP14 and sound a rapid warning beep if the water level exceeds 55000 (critical overflow alarm).
2. **Auto Override Button**: Connect a button on GP13 to manually toggle the pump ON/OFF regardless of the water level.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Pump relay cycles ON and OFF rapidly | Thresholds set too close | Expand the gap between `PUMP_ON_LIMIT` and `PUMP_OFF_LIMIT` to prevent sensor noise from chattering the relay. |
| Pump never turns ON | Sensor signal open | Check the wiring between the sensor OUT pin and GP26, and print raw ADC values in the terminal to verify readings. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [41 - Pico Water Level Serial](41-pico-water-level-serial.md)
- [62 - Pico Relay Button](62-pico-relay-button.md)
