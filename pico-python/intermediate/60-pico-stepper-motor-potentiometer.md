# 60 - Pico Stepper Motor Potentiometer

Control the speed and direction of a stepper motor using a potentiometer.

## Goal
Learn how to generate step pulse sequences, handle digital direction lines, and map analog inputs to step delays in MicroPython.

## What You Will Build
A stepper motor control console:
- **Potentiometer (GP26)**: Middle position stops the motor. Turning left spins the motor counterclockwise (speed increases with rotation). Turning right spins the motor clockwise.
- **ULN2003 Driver & Stepper (GP10-13)**: Drives the stepper motor.

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

step_index = 0

def step_motor(step):
    for pin_idx in range(4):
        pins[pin_idx].value(step_sequence[step][pin_idx])

print("Stepper control console active.")

while True:
    raw = pot.read_u16()
    
    # Check direction and speed relative to center point (32768)
    # Right of center (34000 to 65535) -> Clockwise rotation
    # Left of center (0 to 31000) -> Counterclockwise rotation
    # Center (31000 to 34000) -> Stop
    
    if raw > 34000:
        # Scale speed: larger value -> faster step rate (smaller delay)
        # Delay range: 2 ms (fastest) to 20 ms (slowest)
        diff = raw - 34000
        delay_ms = 20 - int(diff * 18 / (65535 - 34000))
        
        # Step clockwise
        step_index = (step_index + 1) % 4
        step_motor(step_index)
        utime.sleep_ms(delay_ms)
    elif raw < 31000:
        # Scale speed: smaller value -> faster step rate (smaller delay)
        # Delay range: 2 ms (fastest) to 20 ms (slowest)
        diff = 31000 - raw
        delay_ms = 20 - int(diff * 18 / 31000)
        
        # Step counterclockwise
        step_index = (step_index - 1) % 4
        step_motor(step_index)
        utime.sleep_ms(delay_ms)
    else:
        # Stop motor (de-energize coils to prevent heat buildup)
        for pin in pins:
            pin.value(0)
        utime.sleep_ms(100)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Potentiometer**, **ULN2003 Driver**, and **Stepper Motor** onto the canvas.
2. Connect Potentiometer to **GP26**. Connect driver IN1-IN4 to **GP10-13**. Connect power and grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the potentiometer slider left and right to control the stepper motor direction and speed.

## Expected Output
```
Stepper control console active.
```

## Expected Canvas Behavior
- The stepper motor shaft component on the canvas rotates clockwise when the potentiometer is turned right, and counterclockwise when turned left.
- The rotation speed changes based on how far the slider is moved from the center.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `step_sequence` | Lists the coil activation order needed to step the motor. |
| `step_index = (step_index + 1) % 4` | Cycles through the step sequence in order (0 → 1 → 2 → 3 → 0). |
| `delay_ms = 20 - ...` | Scales the potentiometer position to adjust the delay between steps. |

## Hardware & Safety Concept: Stepper Motor De-energization
Unlike DC motors, stepper motors consume holding current even when they are not moving, keeping the internal electromagnet coils energized. When holding a position is not required, it is best to set all phase pins to **0 (LOW)**. De-energizing the coils prevents the stepper motor and driver chip from overheating.

## Try This! (Challenges)
1. **Half-Step commutation**: Modify the step sequence list to implement a 8-step sequence, doubling the step resolution.
2. **Step Counter limit**: Add a limit switch on GP9. Stop the motor immediately if the switch is pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor vibrates but does not spin | Phase wires out of order | Verify that driver inputs IN1-IN4 connect to the Pico's pins GP10-GP13 in order. |
| Driver chip becomes hot | Coils energized when stopped | Ensure all stepper control pins are set to 0 when the motor is stopped. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [11 - Pico Potentiometer Serial](11-pico-potentiometer-serial.md)
- [59 - Pico Servo Knob](59-pico-servo-knob.md)
