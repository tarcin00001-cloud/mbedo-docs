# 46 - Pico Flame Sensor Serial

Build a fire safety interface that reads a flame sensor's analog level and digital indicator pins to detect open fires.

## Goal
Learn how to read both digital and analog inputs from a single sensor board in parallel, map values, and log safety states in MicroPython.

## What You Will Build
A fire detector interface:
- **Flame Sensor Analog (GP26)**: Measures fire intensity.
- **Flame Sensor Digital (GP16)**: Triggers when any open flame is detected.
- **Serial Output**: Prints raw analog readings, digital status, and safety alerts to the terminal every second.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Flame Sensor Module | `potentiometer` | Yes (represented by potentiometer slider) | Yes (IR flame board) |
| Tactile Push Button | `button` | Yes (represents digital limit pin) | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Flame Sensor Board | VCC (+) | 3.3V (3V3) | Red | Power line |
| Flame Sensor Board | GND (−) | GND | Black | Ground reference |
| Flame Sensor Board | AO (Analog Out) | GP26 | Yellow | Analog intensity level |
| Flame Sensor Board | DO (Digital Out) | GP16 | Blue | Digital fire trigger |

> **Wiring tip:** Connect the flame board's VCC to 3.3V, GND to GND, AO to GP26, and DO to GP16. In MbedO, the digital trigger is simulated with a button component.

## Code
```python
from machine import Pin, ADC
import utime

flame_analog  = ADC(26)                 # GP26 = ADC channel 0
flame_digital = Pin(16, Pin.IN, Pin.PULL_UP) # GP16 = Digital trigger input

print("Flame detection console armed.")

while True:
    raw_val = flame_analog.read_u16()
    is_fire = (flame_digital.value() == 0) # Active-LOW digital trigger
    
    # Scale analog reading (high values indicate no fire, low values indicate flame)
    # 65535 = Safe, 10000 = Large fire
    intensity = (65535 - raw_val) * 100 // 55535
    if intensity < 0:
        intensity = 0
    elif intensity > 100:
        intensity = 100
        
    print("Fire Status: ", "ALERT! FIRE" if is_fire else "SAFE",
          "| Analog Reading:", raw_val,
          "| Intensity:", intensity, "%")
          
    utime.sleep(1) # 1-second update cycle
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Flame Sensor** (represented by potentiometer), and **Push Button** (representing digital limit) onto the canvas.
2. Connect Flame Sensor wiper to **GP26**. Connect Button to **GP16**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the analog sensor down and click the digital button to simulate fire events.

## Expected Output
```
Flame detection console armed.
Fire Status:  SAFE | Analog Reading: 65535 | Intensity: 0 %
Fire Status:  ALERT! FIRE | Analog Reading: 30000 | Intensity: 64 %
```

## Expected Canvas Behavior
- The serial terminal prints safety alerts when the analog slider is moved down and the digital button is clicked.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `flame_digital.value() == 0` | Checks the digital comparator output pin (LOW indicates open fire detected). |
| `(65535 - raw_val) * 100 // ...` | Scales and inverts the analog value so that a larger flame results in a higher intensity percentage. |

## Hardware & Safety Concept: Infrared Phototransistors
Flame sensors use an **infrared phototransistor** sensitive to light wavelengths between **760 nm and 1100 nm** (typically emitted by open flames). Since sunlight and incandescent light bulbs also emit infrared light, flame sensors should be shielded or placed away from direct sunlight to prevent false alarms.

## Try This! (Challenges)
1. **Fire Suppression Relay**: Connect an LED/Relay on GP15 that turns ON (simulating a water valve or fire pump) when a flame is detected.
2. **Audio Warning Alarm**: Connect a buzzer on GP14 to sound a pulsing siren when a fire is detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Digital status is always FIRE | Threshold set too sensitive | On real hardware, rotate the onboard potentiometer screw on the sensor breakout board to adjust the digital trigger sensitivity. |
| Intensity reads 0% when fire is present | Sensor out of range | Ensure the flame is within the sensor's viewing angle (typically 60 degrees) and range (up to 1 meter). |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [10 - Pico LDR Light Sensor Serial](10-pico-ldr-serial.md)
- [41 - Pico Water Level Serial](41-pico-water-level-serial.md)
- [43 - Pico Rain Detector Serial](43-pico-rain-detector-serial.md)
