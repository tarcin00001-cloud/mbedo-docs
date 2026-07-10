# 43 - Pico Rain Detector Serial

Build a weather monitoring interface that reads a rain sensor's analog level and digital indicator pins to detect incoming storms.

## Goal
Learn how to read both digital and analog inputs from a single sensor board in parallel, map values, and log weather states in MicroPython.

## What You Will Build
A meteorological rain sensor:
- **Rain Sensor Analog (GP26)**: Measures rain intensity.
- **Rain Sensor Digital (GP16)**: Triggers when any water droplet is detected.
- **Serial Output**: Prints raw analog readings, digital status, and weather alerts to the terminal every second.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Rain Sensor Module | `potentiometer` | Yes (represented by potentiometer slider) | Yes (resistive rain board) |
| Tactile Push Button | `button` | Yes (represents digital limit pin) | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Rain Sensor Board | VCC (+) | 3.3V (3V3) | Red | Power line |
| Rain Sensor Board | GND (−) | GND | Black | Ground reference |
| Rain Sensor Board | AO (Analog Out) | GP26 | Yellow | Analog intensity level |
| Rain Sensor Board | DO (Digital Out) | GP16 | Blue | Digital droplet trigger |

> **Wiring tip:** Connect the rain board's VCC to 3.3V, GND to GND, AO to GP26, and DO to GP16. In MbedO, the digital trigger is simulated with a button component.

## Code
```python
from machine import Pin, ADC
import utime

rain_analog  = ADC(26)                 # GP26 = ADC channel 0
rain_digital = Pin(16, Pin.IN, Pin.PULL_UP) # GP16 = Digital trigger input

print("Rain detection console armed.")

while True:
    raw_val = rain_analog.read_u16()
    is_raining = (rain_digital.value() == 0) # Active-LOW digital trigger
    
    # Scale analog reading (high values indicate dry conditions, low values indicate wet)
    # 65535 = Dry, 10000 = Heavy rain
    intensity = (65535 - raw_val) * 100 // 55535
    if intensity < 0:
        intensity = 0
    elif intensity > 100:
        intensity = 100
        
    print("Rain Status: ", "WET/RAIN" if is_raining else "DRY",
          "| Analog Reading:", raw_val,
          "| Intensity:", intensity, "%")
          
    utime.sleep(1) # 1-second update cycle
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Rain Sensor** (represented by potentiometer), and **Push Button** (representing digital limit) onto the canvas.
2. Connect Rain Sensor wiper to **GP26**. Connect Button to **GP16**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the analog sensor down and click the digital button to simulate rain events.

## Expected Output
```
Rain Status:  DRY | Analog Reading: 65535 | Intensity: 0 %
Rain Status:  WET/RAIN | Analog Reading: 30000 | Intensity: 64 %
```

## Expected Canvas Behavior
- The serial terminal prints weather alerts when the analog slider is moved down and the digital button is clicked.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `rain_digital.value() == 0` | Checks the digital comparator output pin (LOW indicates rain droplets detected). |
| `(65535 - raw_val) * 100 // ...` | Scales and inverts the analog value so that heavier rain results in a higher intensity percentage. |

## Hardware & Safety Concept: Sensor Comparators
Many sensor modules (like rain, sound, and gas sensors) include a built-in **comparator chip** (e.g. LM393) on their breakout boards. This chip compares the analog sensor voltage with a threshold set by a small onboard trim potentiometer. The digital output pin (DO) triggers immediately when the threshold is crossed, allowing microcontrollers to detect events without needing to perform continuous analog-to-digital conversions.

## Try This! (Challenges)
1. **Window Closer Relay**: Connect an LED/Relay on GP15 that turns ON (simulating a window actuator closing) when rain is detected.
2. **Audio Warning Alarm**: Connect a buzzer on GP14 to sound a beep when the first rain droplets are detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Digital status is always DRY | Trim potentiometer not adjusted | On real hardware, rotate the onboard potentiometer screw on the sensor breakout board to adjust the digital trigger sensitivity. |
| Intensity reads 0% when wet | Calibration bounds incorrect | Check the raw analog readings in wet conditions and adjust the scaling offset in your code. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [10 - Pico LDR Light Sensor Serial](10-pico-ldr-serial.md)
- [41 - Pico Water Level Serial](41-pico-water-level-serial.md)
- [42 - Pico Soil Moisture Serial](42-pico-soil-moisture-serial.md)
