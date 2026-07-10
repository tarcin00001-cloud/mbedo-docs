# 01 - Pico Blink

Blink the onboard LED of the Raspberry Pi Pico.

## Goal
Learn how to configure a GPIO pin as a digital output on the RP2040 chip and toggle its state using simple delays.

## What You Will Build
A flashing light indicator:
- **Onboard LED**: The built-in LED on the Pico (GP25) cycles between ON (HIGH) and OFF (LOW) states every 500 milliseconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |

## Wiring
No external wiring is required as this project utilizes the internal onboard LED.

## Code
```cpp
const int LED_PIN = 25; // Onboard LED GP25

void setup() {
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  digitalWrite(LED_PIN, HIGH);
  delay(500);
  digitalWrite(LED_PIN, LOW);
  delay(500);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** onto the canvas.
2. Paste the C++ code into the code editor.
3. Click the **Run** button.
4. Watch the built-in green LED on the Pico board flash.

## Expected Output

Terminal:
```
Simulation active. GP25 toggling.
```

## Expected Canvas Behavior
| Time (ms) | LED Pin (GP25) State | Onboard LED Visual |
| --- | --- | --- |
| 0–500 | HIGH | Green light ON |
| 500–1000 | LOW | Green light OFF |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `pinMode(25, OUTPUT)` | Tells the RP2040 chip's pin controller that GP25 should drive current out. |
| `digitalWrite(25, HIGH)` | Drives the pin voltage to 3.3V, lighting up the onboard LED. |
| `delay(500)` | Suspends program execution for 500 ms (0.5 seconds). |

## Hardware & Safety Concept: Current-Limiting Resistors
On real hardware, the onboard LED is pre-wired to a 470-ohm surface-mount resistor on the Pico PCB. This resistor limits the current drawn by the LED from the GPIO pin. Direct connection without a resistor would burn out the LED or damage the RP2040 processor pin.

## Try This! (Challenges)
1. **Heartbeat Double Flash**: Change the delay intervals to create a double-flash heartbeat pattern (e.g. ON 100ms, OFF 100ms, ON 100ms, OFF 600ms).
2. **Slow Blink**: Change the delay times to make the LED stay ON for 2 seconds and OFF for 2 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Onboard LED does not blink | Wrong pin assignment | Ensure `LED_PIN` is set to `25` (GP25) on the Pico, not 13 (which is Uno default). |

## Mode Notes
This basic GPIO project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [02 - Pico Blink Interval](02-pico-blink-interval.md)
- [03 - Pico Multiple Blink](03-pico-multiple-blink.md)
