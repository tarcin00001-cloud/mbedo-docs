# 02 - Pico Blink Interval

Adjust the blink rate of the onboard LED to learn how delay variations affect the visual flashing rate.

## Goal
Learn how to modify constant delay variables to speed up or slow down a digital output cycle on the Raspberry Pi Pico.

## What You Will Build
A fast-pulsing status indicator:
- **Fast Blink**: The onboard LED (GP25) cycles between ON (HIGH) and OFF (LOW) states every 150 milliseconds, creating a rapid flashing effect.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |

## Wiring
No external wiring is required as this project utilizes the internal onboard LED.

## Code
```cpp
const int LED_PIN = 25; // Onboard LED GP25
const int BLINK_DELAY = 150; // Rapid blink delay

void setup() {
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  digitalWrite(LED_PIN, HIGH);
  delay(BLINK_DELAY);
  digitalWrite(LED_PIN, LOW);
  delay(BLINK_DELAY);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** onto the canvas.
2. Paste the C++ code into the code editor.
3. Click the **Run** button.
4. Watch the built-in green LED flash rapidly.

## Expected Output

Terminal:
```
Simulation active. GP25 flashing rapidly at 150ms intervals.
```

## Expected Canvas Behavior
| Time (ms) | LED Pin (GP25) State | Onboard LED Visual |
| --- | --- | --- |
| 0–150 | HIGH | Green light ON |
| 150–300 | LOW | Green light OFF |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `const int BLINK_DELAY = 150` | Defines a constant integer value in milliseconds to set the cycle rate. |
| `delay(BLINK_DELAY)` | Halts execution for 150 ms to hold the current pin state. |

## Hardware & Safety Concept: Pin Switching Speeds
Microcontrollers like the RP2040 can toggle GPIO states in nanoseconds. However, changing pin states at extreme frequencies consumes more current and can cause voltage noise on power rails. Adding a small delay ensures stable operation and makes the flashing rate visible to the human eye.

## Try This! (Challenges)
1. **Asymmetric Blink**: Make the LED stay ON for 100ms and OFF for 900ms to create a low-power indicator blink.
2. **Speed Increment**: Set `BLINK_DELAY` to 50ms and observe how the rapid pulsing appears almost continuous (persistence of vision threshold).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Flash is too fast to see | Delay value set too low | Verify `BLINK_DELAY` is not set below 20 ms. |

## Mode Notes
This basic GPIO project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [01 - Pico Blink](01-pico-blink.md)
- [03 - Pico Multiple Blink](03-pico-multiple-blink.md)
