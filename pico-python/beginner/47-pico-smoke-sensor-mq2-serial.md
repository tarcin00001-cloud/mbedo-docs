# 47 - Pico Smoke Sensor MQ2 Serial

Build a gas safety interface that reads an MQ-2 smoke/combustible gas sensor's analog level and digital indicator pins.

## Goal
Learn how to read both digital and analog inputs from an MQ-2 sensor breakout board in parallel, calculate gas thresholds, and log safety states in MicroPython.

## What You Will Build
A gas safety monitor:
- **MQ-2 Sensor Analog (GP26)**: Measures gas/smoke concentrations.
- **MQ-2 Sensor Digital (GP16)**: Triggers when the gas level exceeds safety limits.
- **Serial Output**: Prints raw analog readings, digital status, and safety alerts to the terminal every second.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MQ-2 Gas Sensor Module | `potentiometer` | Yes (represented by potentiometer slider) | Yes (MQ-2 gas board) |
| Tactile Push Button | `button` | Yes (represents digital limit pin) | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MQ-2 Sensor Board | VCC (+) | 5V (VBUS) | Red | MQ sensors require 5V heater power |
| MQ-2 Sensor Board | GND (−) | GND | Black | Ground reference |
| MQ-2 Sensor Board | AO (Analog Out) | GP26 | Yellow | Analog intensity level |
| MQ-2 Sensor Board | DO (Digital Out) | GP16 | Blue | Digital gas trigger |

> **Wiring tip:** Connect the MQ-2 board's VCC to the Pico's 5V VBUS pin (since the sensor heater requires 5V to function), GND to GND, AO to GP26, and DO to GP16.

## Code
```python
from machine import Pin, ADC
import utime

mq2_analog  = ADC(26)                 # GP26 = ADC channel 0
mq2_digital = Pin(16, Pin.IN, Pin.PULL_UP) # GP16 = Digital trigger input

print("Gas safety console armed. Warmup active...")
utime.sleep(2) # Initial warmup period

while True:
    raw_val = mq2_analog.read_u16()
    is_gas = (mq2_digital.value() == 0) # Active-LOW digital trigger
    
    # Scale analog reading (low values indicate clean air, high values indicate gas)
    # 8000 = Clean air, 50000 = Dense smoke
    intensity = (raw_val - 8000) * 100 // 42000
    if intensity < 0:
        intensity = 0
    elif intensity > 100:
        intensity = 100
        
    print("Gas Status: ", "ALERT! GAS/SMOKE" if is_gas else "CLEAN AIR",
          "| Analog Reading:", raw_val,
          "| Concentration:", intensity, "%")
          
    utime.sleep(1) # 1-second update cycle
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MQ-2 Sensor** (represented by potentiometer), and **Push Button** (representing digital limit) onto the canvas.
2. Connect MQ-2 wiper to **GP26**. Connect Button to **GP16**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the analog sensor up and click the digital button to simulate gas/smoke events.

## Expected Output
```
Gas safety console armed. Warmup active...
Gas Status:  CLEAN AIR | Analog Reading: 8000 | Concentration: 0 %
Gas Status:  ALERT! GAS/SMOKE | Analog Reading: 35000 | Concentration: 64 %
```

## Expected Canvas Behavior
- The serial terminal prints safety alerts when the analog slider is moved up and the digital button is clicked.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `mq2_digital.value() == 0` | Checks the digital comparator output pin (LOW indicates hazardous gas detected). |
| `(raw_val - 8000) * 100 // ...` | Scales the raw analog value into a gas concentration percentage. |

## Hardware & Safety Concept: MQ Sensor Warmup and Heaters
MQ series gas sensors contain an internal **heating element** that warms a tin dioxide ($SnO_2$) semiconductor layer. When combustible gases contact the heated layer, the sensor's resistance drops. Because the heater takes time to stabilize, MQ sensors require a **warmup period** (typically 20 seconds to 24 hours depending on accuracy requirements) before readings become reliable.

## Try This! (Challenges)
1. **Exhaust Fan Relay**: Connect an LED/Relay on GP15 that turns ON (simulating an exhaust fan) when gas is detected.
2. **Audio Warning Alarm**: Connect a buzzer on GP14 to sound a pulsing warning beep when gas is detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sensor reads high values constantly on boot | Warmup phase active | Allow the sensor to run for a few minutes to heat up and stabilize. |
| Digital status is always GAS | Threshold set too sensitive | On real hardware, rotate the onboard potentiometer screw on the sensor breakout board to adjust the digital trigger sensitivity. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [10 - Pico LDR Light Sensor Serial](10-pico-ldr-serial.md)
- [41 - Pico Water Level Serial](41-pico-water-level-serial.md)
- [46 - Pico Flame Sensor Serial](46-pico-flame-sensor-serial.md)
