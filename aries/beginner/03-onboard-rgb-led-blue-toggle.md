# 03 - Onboard RGB LED Blue toggle

Toggle the onboard RGB LED's Blue channel on the VEGA ARIES v3 board at regular intervals.

## Goal
Learn how to define onboard pins, initialize digital output pins for the Blue LED channel, and execute toggle transitions using delay commands.

## What You Will Build
An onboard RGB LED's Blue channel connected to pin `LED_B` (GPIO 25) is toggled ON and OFF every 1500 milliseconds, creating a slow beacon flashing indicator.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Onboard RGB LED | Built-in | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Onboard LED (Blue) | LED_B | LED_B (GPIO 25) | Internal | No external wiring required |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The onboard RGB LED channels are pre-routed on the VEGA ARIES v3 PCB, so no external breadboard connections are needed.

## Code
```cpp
// Onboard RGB LED Blue Toggle - VEGA ARIES v3
void setup() {
  // Initialize the Blue channel of the onboard RGB LED as an output
  pinMode(LED_B, OUTPUT);
}

void loop() {
  // Turn the Blue LED ON
  digitalWrite(LED_B, HIGH);
  delay(1500);
  
  // Turn the Blue LED OFF
  digitalWrite(LED_B, LOW);
  delay(1500);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board onto the canvas.
2. Select the board and verify that the onboard components are visible.
3. Paste the code into the editor.
4. Click **Run**.
5. Observe the Blue LED widget on the board flashing at 1.5-second intervals.

## Expected Output
Serial Monitor:
```
System Initialized.
GPIO 25 (LED_B) State: HIGH
GPIO 25 (LED_B) State: LOW
```

## Expected Canvas Behavior
* The Blue channel of the onboard RGB LED lights up for 1.5 seconds, dims for 1.5 seconds, and repeats.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(LED_B, OUTPUT)` | Configures the Blue LED pin as a digital output. |
| `digitalWrite(LED_B, HIGH)` | Drives the Blue LED pin to a logic HIGH state (turning it ON). |
| `delay(1500)` | Suspends program execution for 1500 milliseconds (1.5 seconds). |

## Hardware & Safety Concept: THEJAS32 SoC and Onboard LED Routing
* **THEJAS32 SoC**: Built on the 32-bit RISC-V architecture, the THEJAS32 microcontroller handles hardware pin toggling. The onboard pins `LED_R`, `LED_G`, and `LED_B` are internally mapped to specific GPIO lines.
* **Onboard LED Routing**: The Blue LED channel is pre-connected to GPIO 25. Since these channels are active-HIGH in the MbedO simulation, writing `HIGH` turns the LED ON, and writing `LOW` turns it OFF.

## Try This! (Challenges)
1. **Pulsing Beacon**: Change the ON delay to 100 milliseconds and the OFF delay to 2000 milliseconds to simulate a marine beacon.
2. **Variable Timing**: Shorten the delays to 200 milliseconds to create a fast status flash.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED does not light up | Board model mismatch | Ensure you have dragged the correct VEGA ARIES v3 model onto the canvas |
| Compile error on `LED_B` | Capitalization error | Ensure `LED_B` is written in all uppercase letters |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [02 - Onboard RGB LED Green toggle](02-onboard-rgb-led-green-toggle.md)
- [04 - RGB LED Color Mixing (alternating delays)](04-rgb-led-color-mixing.md) (Next project)
