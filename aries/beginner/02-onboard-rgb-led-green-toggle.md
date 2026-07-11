# 02 - Onboard RGB LED Green toggle

Toggle the onboard RGB LED's Green channel on the VEGA ARIES v3 board at regular intervals.

## Goal
Learn how to define onboard pins, initialize digital output pins for the Green LED channel, and execute toggle transitions using delay commands.

## What You Will Build
An onboard RGB LED's Green channel connected to pin `LED_G` (GPIO 24) is toggled ON and OFF every 500 milliseconds, creating a flashing status indicator.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Onboard RGB LED | Built-in | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Onboard LED (Green) | LED_G | LED_G (GPIO 24) | Internal | No external wiring required |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The onboard RGB LED channels are pre-routed on the VEGA ARIES v3 PCB, so no external breadboard connections are needed.

## Code
```cpp
// Onboard RGB LED Green Toggle - VEGA ARIES v3
void setup() {
  // Initialize the Green channel of the onboard RGB LED as an output
  pinMode(LED_G, OUTPUT);
}

void loop() {
  // Turn the Green LED ON
  digitalWrite(LED_G, HIGH);
  delay(500);
  
  // Turn the Green LED OFF
  digitalWrite(LED_G, LOW);
  delay(500);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board onto the canvas.
2. Select the board and verify that the onboard components are visible.
3. Paste the code into the editor.
4. Click **Run**.
5. Observe the Green LED widget on the board flashing at 0.5-second intervals.

## Expected Output
Serial Monitor:
```
System Initialized.
GPIO 24 (LED_G) State: HIGH
GPIO 24 (LED_G) State: LOW
```

## Expected Canvas Behavior
* The Green channel of the onboard RGB LED lights up for 500 milliseconds, dims for 500 milliseconds, and repeats.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(LED_G, OUTPUT)` | Configures the Green LED pin as a digital output. |
| `digitalWrite(LED_G, HIGH)` | Drives the Green LED pin to a logic HIGH state (turning it ON). |
| `delay(500)` | Suspends program execution for 500 milliseconds. |

## Hardware & Safety Concept: THEJAS32 SoC and Onboard LED Routing
* **THEJAS32 SoC**: Built on the 32-bit RISC-V architecture, the THEJAS32 microcontroller handles hardware pin toggling. The onboard pins `LED_R`, `LED_G`, and `LED_B` are internally mapped to specific GPIO lines.
* **Onboard LED Routing**: The Green LED channel is pre-connected to GPIO 24. Since these channels are active-HIGH in the MbedO simulation, writing `HIGH` turns the LED ON, and writing `LOW` turns it OFF.

## Try This! (Challenges)
1. **Heartbeat Pattern**: Create a double-flash pattern by executing two brief flashes followed by a 1-second delay.
2. **Variable Timing**: Shorten the delays to 100 milliseconds to simulate a high-frequency status flash.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED does not light up | Board model mismatch | Ensure you have dragged the correct VEGA ARIES v3 model onto the canvas |
| Compile error on `LED_G` | Capitalization error | Ensure `LED_G` is written in all uppercase letters |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [01 - Onboard RGB LED Blink (Red channel)](01-onboard-rgb-led-blink-red.md)
- [03 - Onboard RGB LED Blue toggle](03-onboard-rgb-led-blue-toggle.md) (Next project)
