# 41 - Pico Water Level Serial

Read water depth level data from an analog sensor and print raw measurements to the serial monitor.

## Goal
Learn how to use `machine.ADC` to read analog voltage inputs from a water level sensor, scale the readings, and log the data to the serial console in MicroPython.

## What You Will Build
A water depth monitor:
- **Water Level Sensor (GP26)**: Measures water contact height.
- **Serial Output**: Prints raw analog readings and estimated depth percentage to the terminal every second.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Water Level Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes (resistive sensor) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Water Level Sensor | VCC (+) | 3.3V (3V3) | Red | Power line |
| Water Level Sensor | GND (−) | GND | Black | Ground reference |
| Water Level Sensor | OUT (Analog) | GP26 | Yellow | Analog input signal |

> **Wiring tip:** Resistive water level sensors output an analog voltage proportional to the immersed length. Connect the sensor's VCC to 3.3V, GND to GND, and OUT to GP26.

## Code
```python
from machine import ADC
import utime

water_sensor = ADC(26) # GP26 = ADC channel 0

# Sensor thresholds (change based on your sensor calibration)
SENSOR_MIN = 8000  # Minimum dry reading
SENSOR_MAX = 52000 # Maximum fully submerged reading

print("Water level monitor online.")

while True:
    raw = water_sensor.read_u16()
    
    # Calculate depth percentage
    percent = (raw - SENSOR_MIN) * 100 // (SENSOR_MAX - SENSOR_MIN)
    
    # Clamp percentage bounds between 0 and 100
    if percent < 0:
        percent = 0
    elif percent > 100:
        percent = 100
        
    print("Raw ADC:", raw, "| Depth:", percent, "%")
    utime.sleep(1) # 1-second update cycle
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Water Level Sensor** (represented by potentiometer) onto the canvas.
2. Connect Sensor VCC to **3.3V**, GND to **GND**, and OUT to **GP26**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the sensor wiper value on the canvas and watch the terminal print depth percentages.

## Expected Output
```
Water level monitor online.
Raw ADC: 5000 | Depth: 0 %
Raw ADC: 30000 | Depth: 50 %
Raw ADC: 52000 | Depth: 100 %
```

## Expected Canvas Behavior
- The serial terminal prints increasing depth percentages as you slide the sensor value up, and decreases to 0% when you slide it down.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `water_sensor.read_u16()` | Reads the analog sensor voltage as a 16-bit integer (0–65535). |
| `(raw - SENSOR_MIN) * 100 // ...` | Linearly scales the calibrated raw range to a user-friendly 0% to 100% depth percentage. |

## Hardware & Safety Concept: Liquid Depth Sensors
Resistive water level sensors use exposed parallel conductive copper traces on a PCB. When dipped in water, the traces are bridged by the liquid, changing the sensor's overall resistance and output voltage. To prevent copper corrosion (electrolysis), real-world systems only power the sensor pin immediately before reading, and keep it turned OFF between updates.

## Try This! (Challenges)
1. **Critical Overflow Alarm**: Connect a buzzer on GP15 and sound an alarm beep if the depth exceeds 90%.
2. **Dynamic Log rate**: Increase the update speed to every 200 ms when the depth is changing rapidly.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Depth reads 0% when wet | Sensor min limit set too high | Check the raw ADC value printed when dry and update `SENSOR_MIN` in your code. |
| Depth reads 100% when dry | Output signal shorted | Verify that the OUT signal pin is connected to GP26 and not shorted directly to the 3.3V rail. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [11 - Pico Potentiometer Serial](11-pico-potentiometer-serial.md)
- [32 - Pico Brightness Level Indicator](32-pico-brightness-level-indicator.md)
