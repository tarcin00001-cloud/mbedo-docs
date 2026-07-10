# 40 - Pico Alarm System Button Buzzer

Build a latching panic alarm system that sounds a continuous buzzer siren when a button is pressed, and remains ON until a separate reset button is pressed.

## Goal
Learn how to implement a software-latching alarm state (Set-Reset latch) using multiple digital inputs to control an active buzzer siren in MicroPython.

## What You Will Build
A latching security alarm:
- **Panic Button (GP14)**: Triggers the alarm state (active LOW).
- **Reset Button (GP13)**: Disarms/clears the alarm state (active LOW).
- **Active Buzzer (GP15)**: Emits a continuous pulsing siren when the alarm is triggered.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Tactile Push Buttons × 2 | `button` | Yes (two buttons) | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Panic Button | Terminal 1 | GP14 | Blue | Set input (trigger alarm) |
| Panic Button | Terminal 2 | GND | Black | Shorts GP14 to GND on press |
| Reset Button | Terminal 1 | GP13 | White | Reset input (disarm alarm) |
| Reset Button | Terminal 2 | GND | Black | Shorts GP13 to GND on press |
| Active Buzzer | VCC (+) | GP15 | Red | Alarm siren output |
| Active Buzzer | GND (−) | GND | Black | Shared return path to ground |

> **Wiring tip:** Both buttons connect their Terminal 2 pins to GND and use internal pull-up resistors on the Pico. Connect the active buzzer to GP15 and GND.

## Code
```python
from machine import Pin
import utime

btn_panic = Pin(14, Pin.IN, Pin.PULL_UP) # Trigger Button
btn_reset = Pin(13, Pin.IN, Pin.PULL_UP) # Reset Button
buzzer    = Pin(15, Pin.OUT)

alarm_triggered = False
buzzer.value(0)

print("Panic alarm system armed.")

while True:
    panic_state = btn_panic.value()
    reset_state = btn_reset.value()
    
    # 1. Check for panic trigger (Set)
    if panic_state == 0 and not alarm_triggered:
        alarm_triggered = True
        print(">> PANIC ALERT TRIGGERED! Siren active.")
        
    # 2. Check for disarm reset (Reset)
    if reset_state == 0 and alarm_triggered:
        alarm_triggered = False
        buzzer.value(0) # Silence buzzer immediately
        print(">> Alarm disarmed and reset.")
        utime.sleep(1) # Lockout delay after reset
        
    # 3. Sound the siren if triggered
    if alarm_triggered:
        buzzer.value(1)
        utime.sleep_ms(150)
        buzzer.value(0)
        utime.sleep_ms(150)
    else:
        utime.sleep_ms(50) # Normal idle loop speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two Push Buttons**, and **Active Buzzer** onto the canvas.
2. Connect Panic Button to **GP14**, Reset Button to **GP13**, and Buzzer to **GP15**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the Panic Button to start the alarm, then click the Reset Button to silence it.

## Expected Output
```
Panic alarm system armed.
>> PANIC ALERT TRIGGERED! Siren active.
>> Alarm disarmed and reset.
```

## Expected Canvas Behavior
- Clicking the Panic Button causes the buzzer component to pulse active continuously.
- Clicking the Reset Button stops the buzzer pulses immediately.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `alarm_triggered = True` | Latches the alarm state to `True`, so it remains active even if the panic button is released. |
| `alarm_triggered = False` | Clears the alarm state to silences the siren when the reset button is pressed. |

## Hardware & Safety Concept: Latching Safety Systems
Industrial safety switches and fire alarm pull stations use mechanical or software latching. Once triggered, the alarm state must remain active until an authorized operator physically resets the panel, ensuring that emergency events cannot go unnoticed or clear themselves automatically.

## Try This! (Challenges)
1. **Warning LED**: Add a Red LED on GP12 that flashes in sync with the buzzer siren.
2. **Delayed Reset**: Modify the code to require holding the reset button down for 3 seconds to clear the alarm state, preventing accidental resets.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers immediately on boot | Panic button wired to 3.3V | Ensure the panic button Terminal 2 is connected to GND, not the 3.3V rail. |
| Reset button does not clear alarm | Reset button pin mapping mismatch | Verify the reset button is wired to GP13 as configured in the code. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [08 - Pico Button Buzzer](08-pico-button-buzzer.md)
- [14 - Pico Dual Button LED](14-pico-dual-button-led.md)
- [20 - Pico Doorbell LED](20-pico-doorbell-led.md)
