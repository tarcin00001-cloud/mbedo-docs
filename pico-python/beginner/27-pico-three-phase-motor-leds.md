# 27 - Pico Three-Phase Motor LEDs

Simulate a three-phase motor phase cycle sequence using three indicator LEDs to represent active winding signals.

## Goal
Learn how to create a overlapping phase-shifted timing sequence on multiple digital output channels to simulate industrial alternating motor driver cycles in MicroPython.

## What You Will Build
A three-phase cycle simulator:
- **LED A (GP13)**: Represents Phase A (Normally cycles ON for 200ms, then OFF).
- **LED B (GP14)**: Represents Phase B (ON after Phase A).
- **LED C (GP15)**: Represents Phase C (ON after Phase B).
- **Phase Sequence**: A (ON) → B (ON) → C (ON) in a rotating cycle to simulate electromagnetic rotation.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Red, Yellow, Green LEDs | `led` | Yes | Yes (represents phases A, B, C) |
| 330 Ω Resistors × 3 | `resistor` | Optional in MbedO | Yes — one per LED |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED A (Red) | Anode (+, longer leg) | GP13 | Red | Phase A output |
| LED B (Yellow) | Anode (+, longer leg) | GP14 | Yellow | Phase B output |
| LED C (Green) | Anode (+, longer leg) | GP15 | Green | Phase C output |
| All Cathodes | Cathode (−) | GND | Black | Shared return path to ground |
| 330 Ω Resistors | Either leg | In series with LED Anodes | — | Limits LED current |

> **Wiring tip:** Connect the three LED anodes to GP13, GP14, and GP15 respectively through 330 Ω resistors. Connect all cathodes to the shared ground rail.

## Code
```python
from machine import Pin
import utime

phase_a = Pin(13, Pin.OUT)
phase_b = Pin(14, Pin.OUT)
phase_c = Pin(15, Pin.OUT)

# Windings cycle rate in milliseconds
STEP_TIME_MS = 250

def run_cycle(a, b, c):
    phase_a.value(a)
    phase_b.value(b)
    phase_c.value(c)
    utime.sleep_ms(STEP_TIME_MS)

print("Starting three-phase sequence simulator.")

while True:
    # 6-step commutation sequence representing rotating magnetic fields
    run_cycle(1, 0, 0) # Phase A active
    run_cycle(1, 1, 0) # Phase A & B overlapping
    run_cycle(0, 1, 0) # Phase B active
    run_cycle(0, 1, 1) # Phase B & C overlapping
    run_cycle(0, 0, 1) # Phase C active
    run_cycle(1, 0, 1) # Phase C & A overlapping
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **three LEDs** onto the canvas.
2. Connect LED A to **GP13**, LED B to **GP14**, and LED C to **GP15**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Observe the overlapping cycling pattern.

## Expected Output
```
Starting three-phase sequence simulator.
```

## Expected Canvas Behavior
- The three LED components on the canvas glow and transition in an overlapping chase sequence (A → AB → B → BC → C → CA → A).

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `run_cycle(1, 1, 0)` | Activates Phase A and B simultaneously for a step to represent overlapping electromagnetic fields. |
| `utime.sleep_ms(STEP_TIME_MS)` | Sets the step duration to define the speed of the simulated rotation. |

## Hardware & Safety Concept: Electromagnetic Commutation
Brushless DC (BLDC) motors and three-phase AC motors rely on sequential, phase-shifted signals to generate rotating magnetic fields inside the stator. This rotating field pulls the magnetic rotor along with it, causing it to spin. Using LED indicator panels helps developers visualize commutation sequences before connecting real high-current motor winding drivers.

## Try This! (Challenges)
1. **Direction Switch**: Add a slide switch on GP12. When flipped, reverse the commutation sequence to reverse the motor's rotation direction (A → C → B).
2. **Speed Potentiometer**: Connect a potentiometer on GP26 to dynamically adjust the speed of the phase commutation cycle.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LEDs turn ON and OFF out of order | Wiring pins swapped | Check that Phase A is on GP13, Phase B is on GP14, and Phase C is on GP15. |
| LEDs flash too fast to see the cycle | Step time set too low | Verify `STEP_TIME_MS` is set to a readable speed (e.g. 250 ms). |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [02 - Pico Double Blink](02-pico-double-blink.md)
- [09 - Pico Traffic Light](09-pico-traffic-light.md)
