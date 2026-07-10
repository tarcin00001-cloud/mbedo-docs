# 28 - Pico Dual Potentiometer Comparer

Compare the voltage levels of two independent potentiometers, and light up an LED when their inputs match.

## Goal
Learn how to read two separate analog input channels (ADC) in parallel, calculate their absolute difference, and perform threshold actions in MicroPython.

## What You Will Build
An analog voltage balance detector:
- **Potentiometer A (GP26)**: Left voltage source.
- **Potentiometer B (GP27)**: Right voltage source.
- **LED (GP15)**: Lights up when both potentiometer knobs are adjusted to approximately the same angle (within a small tolerance).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Potentiometers × 2 | `potentiometer` | Yes (two potentiometers) | Yes |
| LED (any colour) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer A | Left Pin (Pin 1) | GND | Black | Ground reference |
| Potentiometer A | Wiper (Pin 2 - centre) | GP26 | Yellow | Analog input A |
| Potentiometer A | Right Pin (Pin 3) | 3.3V (3V3) | Red | Reference voltage |
| Potentiometer B | Left Pin (Pin 1) | GND | Black | Ground reference |
| Potentiometer B | Wiper (Pin 2 - centre) | GP27 | White | Analog input B |
| Potentiometer B | Right Pin (Pin 3) | 3.3V (3V3) | Red | Reference voltage |
| LED | Anode (+, longer leg) | GP15 | Orange | Balance indicator |
| LED | Cathode (−, shorter leg) | GND | Black | Shared return path to ground |
| 330 Ω Resistor | Either leg | In series with LED Anode | — | Limits LED current |

> **Wiring tip:** Connect the wipers of Potentiometer A and B to GP26 and GP27 respectively. Connect both left pins to GND, and both right pins to 3.3V. Connect the LED anode to GP15.

## Code
```python
from machine import Pin, ADC
import utime

pot_a = ADC(26) # GP26 = ADC channel 0
pot_b = ADC(27) # GP27 = ADC channel 1
led   = Pin(15, Pin.OUT)

# Allowed margin of difference (out of 65535 raw scale)
MATCH_TOLERANCE = 3000

while True:
    val_a = pot_a.read_u16()
    val_b = pot_b.read_u16()
    
    # Calculate absolute difference
    diff = val_a - val_b
    if diff < 0:
        diff = -diff
        
    print("A:", val_a, "| B:", val_b, "| Diff:", diff)
    
    # Turn ON LED if values are balanced (close match)
    if diff < MATCH_TOLERANCE:
        led.value(1) # Balanced
        print(">> Inputs balanced!")
    else:
        led.value(0) # Unbalanced
        
    utime.sleep_ms(200) # Polling update delay
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two Potentiometers**, and **LED** onto the canvas.
2. Connect Potentiometer A to **GP26**, Potentiometer B to **GP27**, and LED to **GP15**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide both potentiometer knobs to the same relative position and watch the LED turn ON.

## Expected Output
```
A: 32000 | B: 12000 | Diff: 20000
A: 32000 | B: 31000 | Diff: 1000
>> Inputs balanced!
```

## Expected Canvas Behavior
- The LED component on the canvas lights up only when the two potentiometer sliders are adjusted to matching levels.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `ADC(27)` | Configures GP27 as a second analog input channel to read Potentiometer B. |
| `diff < 0; diff = -diff` | Computes the absolute value of the difference between the two sensor readings. |
| `diff < MATCH_TOLERANCE` | Checks if the difference is within the allowed tolerance margin. |

## Hardware & Safety Concept: Balance Detection
Dual-channel balance detectors are commonly used in safety-critical systems (like dual flight control computers, industrial emergency stops, or drive-by-wire throttles). If the two independent sensors do not report matching values (within a safe tolerance), the controller detects a fault and shuts down operations to prevent safety issues.

## Try This! (Challenges)
1. **Balance Buzzer**: Add an active buzzer on GP14 that sounds a beep when the sensors are unbalanced.
2. **Direction Indicators**: Add two more LEDs: a Red LED on GP13 (ON when A > B) and a Yellow LED on GP12 (ON when B > A).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED never turns ON | Tolerance margin too tight | Increase `MATCH_TOLERANCE` (e.g. to 5000) to account for slight wiper noise or calibration differences. |
| Readings are identical at all times | Both wires connected to the same pin | Check the wiring to ensure Potentiometer A goes to GP26 and Potentiometer B goes to GP27. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [11 - Pico Potentiometer Serial](11-pico-potentiometer-serial.md)
- [17 - Pico LED Brightness Knob](17-pico-led-brightness-knob.md)
