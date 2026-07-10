# 59 - Pico Servo Knob

Control the shaft angle of a servo motor dynamically using a rotary potentiometer.

## Goal
Learn how to read an analog potentiometer value (0–65535) and map it to a matching PWM duty cycle range to steer a servo motor from 0 to 180 degrees in MicroPython.

## What You Will Build
A servo steering console:
- **Potentiometer (GP26)**: Reads knob adjustments.
- **Servo Motor (GP10)**: Rotates its shaft to match the angle of the potentiometer knob.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes (standard SG90) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Left Pin (Pin 1) | GND | Black | Ground reference |
| Potentiometer | Wiper (Pin 2 - centre) | GP26 | Yellow | Analog input pin |
| Potentiometer | Right Pin (Pin 3) | 3.3V (3V3) | Red | High voltage reference |
| Servo Motor | Control (PWM) | GP10 | Orange | PWM angle signal |
| Servo Motor | Power (+) | 5V (VBUS) | Red | Servo power supply (5V) |
| Servo Motor | GND (−) | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the servo's power (+) pin to the Pico's 5V VBUS pin (not 3.3V, as servos need 5V to run under load). Connect the control pin to GP10, and GND to GND. Connect the potentiometer wiper pin to GP26.

## Code
```python
from machine import Pin, ADC, PWM
import utime

pot = ADC(26) # GP26 = ADC channel 0

# Set up Servo on GP10 (servos run on a 50 Hz PWM frequency)
servo = PWM(Pin(10))
servo.freq(50)

# Standard servo pulse widths:
# 0 degrees   = 0.5 ms pulse -> duty = (0.5ms / 20ms) * 65535 = 1638
# 90 degrees  = 1.5 ms pulse -> duty = (1.5ms / 20ms) * 65535 = 4915
# 180 degrees = 2.5 ms pulse -> duty = (2.5ms / 20ms) * 65535 = 8192
MIN_DUTY = 1638
MAX_DUTY = 8192

print("Servo steering console armed.")

while True:
    raw = pot.read_u16()
    
    # Map raw value (0 - 65535) to duty range (MIN_DUTY - MAX_DUTY)
    duty = MIN_DUTY + int(raw * (MAX_DUTY - MIN_DUTY) / 65535)
    
    # Calculate angle for serial readout
    angle = int(raw * 180 / 65535)
    
    # Apply pulse to servo
    servo.duty_u16(duty)
    
    print("Potentiometer:", raw, "| Angle:", angle, "degrees | Duty:", duty)
    utime.sleep_ms(50) # Polling update delay
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Potentiometer**, and **Servo Motor** onto the canvas.
2. Connect Potentiometer wiper to **GP26**. Connect Servo Control to **GP10**, Power to **5V**, and GND to **GND**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the potentiometer slider on the canvas and observe the servo rotating.

## Expected Output
```
Servo steering console armed.
Potentiometer: 0 | Angle: 0 degrees | Duty: 1638
Potentiometer: 32768 | Angle: 90 degrees | Duty: 4915
Potentiometer: 65535 | Angle: 180 degrees | Duty: 8192
```

## Expected Canvas Behavior
- The servo motor shaft component on the canvas rotates in real-time response to the position of the potentiometer slider.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `servo.freq(50)` | Sets the PWM frequency to 50 Hz (20 ms period), the standard signal rate required by analog servo motors. |
| `raw * (MAX_DUTY - MIN_DUTY) / 65535` | Computes the duty cycle scaling offset to control the servo angle. |
| `servo.duty_u16(duty)` | Outputs the scaled duty pulse width to drive the servo motor. |

## Hardware & Safety Concept: Servo Power Supplies
Servo motors contain internal DC motors, gearboxes, and driver circuits. Under load, a servo can draw currents exceeding **1 Ampere**. Trying to power a servo directly from the Pico's 3.3V pin can cause the voltage to drop, causing the Pico to reset (brownout). Always power servos from the 5V VBUS pin (USB power) or an external power supply.

## Try This! (Challenges)
1. **Auto Sweep Mode**: Add a push button on GP12. Pressing it toggles between manual potentiometer steering and automatic sweeps (0 to 180 degrees).
2. **Reverse Steering**: Adjust the mapping formula so that rotating the potentiometer clockwise rotates the servo counterclockwise.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo jitters or makes humming noise | Pulse limits incorrect | Verify that your `MIN_DUTY` and `MAX_DUTY` limits match your servo's physical range. |
| Pico resets when servo moves | Current draw too high | Connect the servo's power (+) pin to the 5V VBUS pin, or use an external 5V power supply. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [11 - Pico Potentiometer Serial](11-pico-potentiometer-serial.md)
- [17 - Pico LED Brightness Knob](17-pico-led-brightness-knob.md)
