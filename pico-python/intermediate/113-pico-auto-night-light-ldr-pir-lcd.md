# 113 - Pico Auto Night Light LDR PIR LCD

Build a smart auto night light that activates a lighting relay when it is dark AND motion is detected, and displays active conditions on an I2C LCD screen.

## Goal
Learn how to combine AND logic between an LDR light sensor and a PIR motion sensor, control a relay output, and display the active system state on an I2C character LCD in MicroPython.

## What You Will Build
An intelligent auto night light:
- **LDR Sensor (GP26)**: Measures ambient light (low reading = dark).
- **PIR Sensor (GP14)**: Detects motion (HIGH output on detection).
- **Relay Module (GP15)**: Turns ON the light ONLY when it is dark AND motion is detected.
- **I2C 16x2 LCD (GP4, GP5)**: Displays the LDR and PIR conditions and relay state.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LDR Photoresistor | `ldr` | Yes | Yes |
| PIR Motion Sensor | `button` | Yes (represented by push button component) | Yes |
| Relay Module | `relay` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | Optional in MbedO | Yes (LDR voltage divider) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR | Pin 1 | 3.3V (3V3) | Red | Power line |
| LDR | Pin 2 | GP26 | Yellow | Analog input signal |
| 10 kΩ Resistor | Either leg | GP26 to GND | Black | Pull-down resistor |
| PIR Sensor | VCC | 5V (VBUS) | Red | PIR power (5V preferred) |
| PIR Sensor | GND | GND | Black | Ground reference |
| PIR Sensor | OUT | GP14 | Blue | Digital motion output (HIGH on motion) |
| Relay Module | IN | GP15 | Orange | Relay control signal |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The LDR and 10 kΩ resistor form a voltage divider connected to GP26. Connect the PIR output to GP14. Connect the relay input to GP15 and the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, ADC, I2C
import utime
from machine_lcd import I2cLcd

ldr         = ADC(26) # GP26 = ADC channel 0
pir         = Pin(14, Pin.IN, Pin.PULL_DOWN)
light_relay = Pin(15, Pin.OUT)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Threshold: below this = dark (raw 0 - 65535)
DARK_THRESHOLD = 20000

light_relay.value(0) # Start with light OFF

lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("Auto Night Light")
lcd.move_to(0, 1)
lcd.putstr("PIR Calibrating")
utime.sleep(2) # PIR sensor warmup

print("Auto night light active.")

while True:
    ldr_raw     = ldr.read_u16()
    motion      = (pir.value() == 1) # HIGH = motion detected
    is_dark     = (ldr_raw < DARK_THRESHOLD)
    
    # AND logic: light ON only when dark AND motion detected
    light_on = is_dark and motion
    light_relay.value(1 if light_on else 0)
    
    # Update LCD Display
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("Lgt:{} Mot:{}".format(
        "DARK" if is_dark else "BRIG",
        "YES " if motion else "NO  "))
    lcd.move_to(0, 1)
    lcd.putstr("Light: " + ("ON " if light_on else "OFF"))
    
    print("Dark:{} | Motion:{} | Light:{}".format(
        is_dark, motion, "ON" if light_on else "OFF"))
    
    utime.sleep_ms(200)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **LDR**, **Push Button** (representing PIR), **Relay**, and **I2C LCD** onto the canvas.
2. Connect LDR to **GP26**, PIR Button to **GP14**, Relay to **GP15**, and LCD to **GP4/GP5**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the LDR down (dark) AND click the PIR button (motion) to activate the relay light.

## Expected Output
```
Auto night light active.
Dark:True | Motion:False | Light:OFF
Dark:True | Motion:True | Light:ON
```
(On screen: "Lgt:DARK Mot:YES" and "Light: ON" when both conditions are met.)

## Expected Canvas Behavior
- The relay component on the canvas closes and the LCD updates to "Light: ON" only when both the LDR slider is below the dark threshold AND the PIR button is clicked simultaneously.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `is_dark = (ldr_raw < DARK_THRESHOLD)` | Evaluates the light condition: `True` if the LDR reading indicates darkness. |
| `light_on = is_dark and motion` | Boolean AND gate: both conditions must be `True` for the light to activate. |

## Hardware & Safety Concept: Conditional AND Logic in Safety Systems
Safety systems use Boolean AND gates to prevent false activations. An auto night light using only a PIR sensor would flash ON during daylight (wasting energy). Adding the LDR as an AND condition ensures the light only activates during actual nighttime hours, regardless of spurious motion triggers from pets or HVAC drafts.

## Try This! (Challenges)
1. **Auto-Off Timer**: Modify the code so the light stays ON for 30 seconds after motion is last detected before turning OFF.
2. **Sensitivity Adjust**: Connect a potentiometer on GP27 to adjust the `DARK_THRESHOLD` dynamically.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Light never turns ON | PIR not calibrated | Wait 30 seconds after power-on for PIR to stabilize. |
| Light turns ON in bright light | Threshold too high | Lower `DARK_THRESHOLD` in your code after measuring the ambient LDR reading. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [22 - Pico Night Light LDR](22-pico-night-light-ldr.md)
- [74 - Pico Motion Security PIR Alarm](74-pico-motion-security-pir-alarm.md)
- [85 - Pico Motion Security PIR LCD](85-pico-motion-security-pir-lcd.md)
