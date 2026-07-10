# 42 - Pico Soil Moisture Serial

Read agricultural soil moisture levels from an analog sensor and log values to the serial monitor.

## Goal
Learn how to use `machine.ADC` to read raw soil moisture levels, calculate volumetric soil content percentages, and print the results to the serial console in MicroPython.

## What You Will Build
A soil moisture monitor:
- **Soil Moisture Sensor (GP26)**: Measures soil conductivity/moisture.
- **Serial Output**: Prints raw analog readings and scaled moisture percentages to the terminal every 2 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Soil Moisture Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes (resistive/capacitive probe) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Soil Sensor | VCC (+) | 3.3V (3V3) | Red | Power line |
| Soil Sensor | GND (−) | GND | Black | Ground reference |
| Soil Sensor | OUT (Analog) | GP26 | Yellow | Analog input signal |

> **Wiring tip:** Connect the soil moisture sensor's VCC to 3.3V, GND to GND, and the analog output signal pin to GP26.

## Code
```python
from machine import ADC
import utime

soil_sensor = ADC(26) # GP26 = ADC channel 0

# Calibration bounds (adjust based on your sensor readings in dry/wet soil)
DRY_VALUE = 48000 # High resistance (Dry soil)
WET_VALUE = 12000 # Low resistance (Water-saturated soil)

print("Soil moisture monitor active.")

while True:
    raw = soil_sensor.read_u16()
    
    # Calculate percentage (invert because wet soil has lower resistance/voltage value)
    percent = (DRY_VALUE - raw) * 100 // (DRY_VALUE - WET_VALUE)
    
    # Clamp bounds
    if percent < 0:
        percent = 0
    elif percent > 100:
        percent = 100
        
    print("Raw reading:", raw, "| Soil Moisture:", percent, "%")
    utime.sleep(2) # 2-second update cycle
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Soil Moisture Sensor** (represented by potentiometer) onto the canvas.
2. Connect Sensor VCC to **3.3V**, GND to **GND**, and OUT to **GP26**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the sensor wiper value on the canvas and watch the terminal print moisture percentages.

## Expected Output
```
Soil moisture monitor active.
Raw reading: 48000 | Soil Moisture: 0 %
Raw reading: 30000 | Soil Moisture: 50 %
Raw reading: 12000 | Soil Moisture: 100 %
```

## Expected Canvas Behavior
- The serial terminal prints increasing moisture percentages as you slide the sensor value down (simulating wet soil), and decreases to 0% when you slide it up (dry soil).

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `soil_sensor.read_u16()` | Reads the analog sensor voltage as a 16-bit integer (0–65535). |
| `(DRY_VALUE - raw) * 100 // ...` | Scales and inverts the reading, since resistive sensors output higher voltages in dry soil. |

## Hardware & Safety Concept: Resistive vs Capacitive Sensors
Resistive soil probes measure electrical resistance between two exposed metal prongs in the dirt. Because electricity flows directly through the soil, resistive probes corrode quickly (oxidation). Capacitive moisture sensors use insulated probes to measure changes in soil capacitance, preventing metal corrosion and providing much longer operational life.

## Try This! (Challenges)
1. **Irrigation Valve Warning**: Connect an LED on GP15 that turns ON when the soil moisture drops below 30% (signaling that watering is needed).
2. **Dynamic Backlight**: Turn OFF the LED automatically when the soil is watered above 60%.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Moisture reads 0% when placed in water | Calibration bounds incorrect | Check the raw ADC value printed when submerged in water and update `WET_VALUE` in your code. |
| Moisture reading fluctuates wildly | Loose connection | Verify that the signal wire is connected firmly to GP26 and not floating. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [11 - Pico Potentiometer Serial](11-pico-potentiometer-serial.md)
- [41 - Pico Water Level Serial](41-pico-water-level-serial.md)
