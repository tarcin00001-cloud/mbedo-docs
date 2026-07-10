# 23 - Pico Light Alarm

Build a security light alarm that sounds a buzzer when a laser line or flashlight beam is broken (sudden drop in light).

## Goal
Learn how to detect sudden relative changes (delta) in analog sensor inputs to trigger digital outputs in MicroPython.

## What You Will Build
An intrusion light alarm:
- **LDR Sensor (GP26)**: Monitors ambient light levels.
- **Active Buzzer (GP15)**: Sounds an alarm when the light level drops suddenly (simulating a broken beam).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LDR Photoresistor | `ldr` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | Optional in MbedO | Yes (voltage divider) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR | Pin 1 | 3.3V (3V3) | Red | Power line |
| LDR | Pin 2 | GP26 | Yellow | Analog signal output |
| 10 kΩ Resistor | Either leg | GP26 to GND | Black | Pull-down resistor |
| Active Buzzer | VCC (+) | GP15 | Red | Alarm output |
| Active Buzzer | GND (−) | GND | Black | Shared return path to ground |

> **Wiring tip:** The LDR and 10 kΩ resistor form a voltage divider at GP26. The active buzzer connects to GP15 and GND.

## Code
```python
from machine import Pin, ADC
import utime

ldr = ADC(26) # GP26 = ADC channel 0
buzzer = Pin(15, Pin.OUT)

buzzer.value(0) # Ensure buzzer starts silent

# Read initial reference level
reference_light = ldr.read_u16()
print("Calibration complete. Reference:", reference_light)

# Trigger limit: alert if light drops by more than 15000 units
THRESHOLD_DROP = 15000

while True:
    current_light = ldr.read_u16()
    
    # Calculate difference
    drop = reference_light - current_light
    
    if drop > THRESHOLD_DROP:
        print("ALARM! Light beam broken. Reading:", current_light)
        # Sound alarm (pulsing beep)
        buzzer.value(1)
        utime.sleep_ms(200)
        buzzer.value(0)
        utime.sleep_ms(200)
    else:
        # Slowly drift reference to adapt to natural lighting changes
        reference_light = int(reference_light * 0.99 + current_light * 0.01)
        buzzer.value(0)
        utime.sleep_ms(100)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **LDR**, and **Active Buzzer** onto the canvas.
2. Connect LDR Pin 1 to **3.3V** and Pin 2 to **GP26**.
3. Connect Buzzer VCC (+) to **GP15** and GND (−) to **GND**.
4. Paste the code, select **MicroPython** mode, and click **Run**.
5. Slide the LDR value down quickly on the canvas to trigger the alarm.

## Expected Output
```
Calibration complete. Reference: 48000
ALARM! Light beam broken. Reading: 20000
```

## Expected Canvas Behavior
- Rapidly sliding the LDR down causes the buzzer component to pulse active, sounding the alarm.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `reference_light - current_light` | Calculates the drop in light intensity compared to the calibrated reference. |
| `reference_light * 0.99 + ...` | Implements a simple low-pass filter to slowly adapt the reference light level to slow ambient changes (like clouds passing or sunset). |

## Hardware & Safety Concept: Light-Beam Intrusion Barriers
Light-beam alarm systems typically use a laser diode focused on a photoresistor. If an intruder steps through the beam, the light level drops instantly, triggering the alarm. Using a relative drop check instead of a fixed threshold prevents the system from false-triggering as the room naturally brightens or dims throughout the day.

## Try This! (Challenges)
1. **Latching Alarm**: Modify the code so the buzzer continues to sound even after the light beam is restored, until a reset button on GP14 is pressed.
2. **Ambient Brightness Alarm**: Sound the alarm if the light level increases suddenly (for example, detecting when a refrigerator door is left open).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers continuously | Reference level drift issue | Ensure the LDR is not physically moving or shaking, which can cause erratic readings. |
| Alarm does not trigger | Threshold drop set too high | Check the ambient light level readings in the terminal and lower the `THRESHOLD_DROP` value if needed. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [10 - Pico LDR Light Sensor Serial](10-pico-ldr-serial.md)
- [22 - Pico Night Light LDR](22-pico-night-light-ldr.md)
