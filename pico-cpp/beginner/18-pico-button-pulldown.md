# 18 - Pico Button Pull-down

Read button inputs using the Pico's internal pull-down resistor.

## Goal
Learn how to configure internal pull-down resistors to establish active-HIGH inputs on the RP2040 chip.

## What You Will Build
An active-HIGH button switch circuit:
- **Push Button (GP16)**: Monitored via internal pull-down. The pin reads `LOW` by default and goes `HIGH` when pressed.
- **Warning LED (GP15)**: Turns ON when the button is pressed (HIGH state).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Push Button | Terminal 1 | GP16 | Sense pin |
| Push Button | Terminal 2 | 3.3V | Connects directly to power |
| Red LED | Anode | GP15 | Control output |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int BUTTON_PIN = 16;
const int LED_PIN    = 15;

void setup() {
  pinMode(BUTTON_PIN, INPUT_PULLDOWN); // Enable internal pull-down resistor
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int buttonState = digitalRead(BUTTON_PIN);

  // Active-HIGH logic: button pressed pulls pin to VCC (HIGH)
  if (buttonState == HIGH) {
    digitalWrite(LED_PIN, HIGH);
  } else {
    digitalWrite(LED_PIN, LOW);
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Push Button**, and **Red LED** onto the canvas.
2. Connect Button: **Terminal 1** to **GP16**, **Terminal 2** to **3.3V** (no external resistor needed!).
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Click and hold the push button to turn on the LED.

## Expected Output

Terminal:
```
Simulation active. Internal pull-down active on GP16.
```

## Expected Canvas Behavior
| Button Input (GP16) | Pin Voltage | LED state (GP15) |
| --- | --- | --- |
| open (Released) | 0V (LOW) | LOW (OFF) |
| Closed (Pressed) | 3.3V (HIGH) | HIGH (ON) |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `pinMode(16, INPUT_PULLDOWN)` | Connects GP16 to internal 50k-ohm pull-down resistor tied to ground inside the RP2040 chip. |
| `buttonState == HIGH` | Checks if the pin has been pulled to 3.3V by the pressed switch contact. |

## Hardware & Safety Concept: Internal Pull-down Limits
While internal resistors simplify circuit assembly, their high resistance (approx. 50k ohms) makes them sensitive to electromagnetic noise in electrically noisy environments. For long wire runs or industrial controls, low-value external resistors (1k or 10k ohms) are preferred to provide stronger pull strength.

## Try This! (Challenges)
1. **Flashing Alert**: Set the LED to blink when the button is held HIGH.
2. **Latch Control**: Program the LED to remain ON once the button is pressed, until the Pico is restarted.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays OFF when pressed | Swapped switch power | Confirm that the switch is wired to the Pico's 3.3V pin, not GND. |

## Mode Notes
This basic GPIO project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [16 - Pico Button LED](16-pico-button-led.md)
- [17 - Pico Button Pull-up](17-pico-button-pullup.md)
