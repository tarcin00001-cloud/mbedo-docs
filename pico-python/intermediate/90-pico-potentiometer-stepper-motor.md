# 90 - Pico Potentiometer Stepper Motor

Control the shaft rotation angle of a stepper motor dynamically using a rotary potentiometer.

## Goal
Learn how to map an analog potentiometer value (0–65535) to a target number of stepper motor steps, generate sequence steps, and drive driver chips in MicroPython.

## What You Will Build
An analog-controlled stepper positioner:
- **Potentiometer (GP26)**: Reads target rotation angle.
- **ULN2003 Driver & Stepper (GP10-13)**: Rotates its shaft to match the angle of the potentiometer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| Stepper Motor (28BYJ-48) | `stepper` | Yes | Yes |
| Stepper Driver (ULN2003) | `stepper_driver` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Left Pin (Pin 1) | GND | Black | Ground reference |
| Potentiometer | Wiper (Pin 2 - centre) | GP26 | Yellow | Analog input pin |
| Potentiometer | Right Pin (Pin 3) | 3.3V (3V3) | Red | High voltage reference |
| ULN2003 Driver | IN1 | GP10 | Orange | Stepper Phase A control |
| ULN2003 Driver | IN2 | GP11 | Yellow | Stepper Phase B control |
| ULN2003 Driver | IN3 | GP12 | Green | Stepper Phase C control |
| ULN2003 Driver | IN4 | GP13 | Blue | Stepper Phase D control |
| ULN2003 Driver | VCC / GND | 5V (VBUS) / GND | Red / Black | Driver power supply |

> **Wiring tip:** Connect the ULN2003 driver inputs IN1-IN4 to GP10-GP13. Connect the driver VCC to the Pico's 5V VBUS pin, and GND to GND. Connect the potentiometer wiper pin to GP26.

## Code
```python
from machine import Pin, ADC
import utime

pot = ADC(26) # GP26 = ADC channel 0

# Set up stepper control pins
pins = [Pin(10, Pin.OUT), Pin(11, Pin.OUT), Pin(12, Pin.OUT), Pin(13, Pin.OUT)]

# 4-step sequence for stepper commutation
step_sequence = [
    [1, 0, 0, 0],
    [0, 1, 0, 0],
    [0, 0, 1, 0],
    [0, 0, 0, 1]
]

# Total steps for a full 360-degree rotation of 28BYJ-48 motor (in 4-step mode)
STEPS_PER_REV = 2048

current_step = 0
step_index = 0

def step_motor(direction):
    global step_index
    if direction == 1:
        step_index = (step_index + 1) % 4
    else:
        step_index = (step_index - 1) % 4
        
    for pin_idx in range(4):
        pins[pin_idx].value(step_sequence[step_index][pin_idx])
    utime.sleep_ms(3) # Speed delay between steps

print("Stepper positioner console active.")

while True:
    raw = pot.read_u16()
    
    # Map raw value (0 - 65535) to target step position (0 - 2047)
    target_step = int(raw * (STEPS_PER_REV - 1) / 65535)
    
    # Move motor toward target step position
    if current_step < target_step:
        step_motor(1) # Step forward
        current_step += 1
        print("Position:", current_step, "/", target_step)
    elif current_step > target_step:
        step_motor(-1) # Step backward
        current_step -= 1
        print("Position:", current_step, "/", target_step)
    else:
        # de-energize coils to prevent heat buildup when stopped
        for pin in pins:
            pin.value(0)
        utime.sleep_ms(20) # Small delay when idling
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Potentiometer**, **ULN2003 Driver**, and **Stepper Motor** onto the canvas.
2. Connect Potentiometer to **GP26**. Connect driver IN1-IN4 to **GP10-13**. Connect power and grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the potentiometer slider on the canvas and observe the stepper motor rotating to matching positions.

## Expected Output
```
Stepper positioner console active.
Position: 120 / 512
Position: 121 / 512
```

## Expected Canvas Behavior
- The stepper motor shaft component on the canvas rotates to match the position of the potentiometer slider.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `raw * (STEPS_PER_REV - 1) / 65535` | Scales the 16-bit ADC input to a target step index between 0 and 2047. |
| `step_motor(1)` | Runs the stepping sequence in the forward direction. |
| `current_step += 1` | Increments the active step tracking variable to match the physical position. |

## Hardware & Safety Concept: Stepper Motor Calibration
Unlike servo motors, standard stepper motors operate in **open loop** mode: they do not have internal encoders to verify their actual shaft position. If the motor encounters an obstruction, it can slip steps and lose calibration. To ensure accuracy, real-world systems include a homing limit switch to recalibrate the zero point on boot.

## Try This! (Challenges)
1. **Calibration Limit Switch**: Connect a button on GP14 to simulate a homing switch. Sweep the stepper backward on boot until the switch is clicked to set the zero reference.
2. **Speed Controller**: Add a second potentiometer on GP27 to adjust the stepping delay speed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor vibrates but does not spin | Phase wires out of order | Verify that driver inputs IN1-IN4 connect to the Pico's pins GP10-GP13 in order. |
| Position drifts over time | Slipping steps | Stepper motor steps can slip if rotating too fast. Increase the step delay from `3` ms to `5` ms. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [11 - Pico Potentiometer Serial](11-pico-potentiometer-serial.md)
- [59 - Pico Servo Knob](59-pico-servo-knob.md)
- [60 - Pico Stepper Motor Potentiometer](60-pico-stepper-motor-potentiometer.md)
