# 01 - Onboard RGB LED Blink (Red channel)

Blink the onboard RGB LED's Red channel on the VEGA ARIES v3 board at regular intervals.

## Goal
Learn how to configure the VEGA ARIES v3 development board in MbedO, define onboard pin outputs, initialize digital pins, and toggle high/low states using standard delay commands.

## What You Will Build
An onboard RGB LED's Red channel connected to pin `LED_R` (GPIO 23) is toggled ON and OFF every 1000 milliseconds, creating a steady blinking indicator.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Onboard RGB LED | Built-in | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Onboard LED (Red) | LED_R | LED_R (GPIO 23) | Internal | No external wiring required |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Since the RGB LED is integrated directly onto the VEGA ARIES v3 board, no external jumper wires or breadboards are required for this project.

## Code
```cpp
// Onboard RGB LED Red Blink - VEGA ARIES v3
void setup() {
  // Initialize the Red channel of the onboard RGB LED as an output
  pinMode(LED_R, OUTPUT);
}

void loop() {
  // Turn the Red LED ON
  digitalWrite(LED_R, HIGH);
  delay(1000);
  
  // Turn the Red LED OFF
  digitalWrite(LED_R, LOW);
  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board onto the canvas.
2. Select the board and verify that the onboard components are visible.
3. Paste the code into the editor.
4. Click **Run**.
5. Observe the Red LED widget on the board blinking at 1-second intervals.

## Expected Output
Serial Monitor:
```
System Initialized.
GPIO 23 (LED_R) State: HIGH
GPIO 23 (LED_R) State: LOW
```

## Expected Canvas Behavior
* The Red channel of the onboard RGB LED lights up for 1 second, dims for 1 second, and repeats.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(LED_R, OUTPUT)` | Configures the Red LED pin as a digital output. |
| `digitalWrite(LED_R, HIGH)` | Drives the Red LED pin to a logic HIGH state (turning it ON). |
| `delay(1000)` | Suspends program execution for 1000 milliseconds (1 second). |

## Hardware & Safety Concept: THEJAS32 SoC and Onboard Current Limits
* **THEJAS32 SoC**: The VEGA ARIES v3 board is powered by the THEJAS32 SoC, an Indian-designed 32-bit RISC-V microcontroller. Its GPIO pins support standard digital logic operations but have strict drive current limitations.
* **Onboard Current Limits**: The onboard RGB LED channels are pre-wired with current-limiting resistors (typically 330 Ω) to protect the SoC pins from drawing excessive current. When wiring external LEDs, always add a matching resistor in series.

## Try This! (Challenges)
1. **Faster Blink Rate**: Change the delay to 250 milliseconds to create a rapid warning pulse.
2. **Double Blink Pattern**: Modify the loop to turn the LED ON and OFF twice quickly, followed by a longer delay.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Onboard LED does not light up | Board model mismatch | Ensure you have dragged the correct VEGA ARIES v3 model, not an Arduino or ESP32 board |
| Compile error on `LED_R` | Spelling error | Ensure `LED_R` is written in all capitals as defined by the board definitions |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [02 - Onboard RGB LED Green toggle](02-onboard-rgb-led-green-toggle.md) (Next project)
