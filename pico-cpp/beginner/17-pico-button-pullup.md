# 17 - Pico Button Pull-up

Read button inputs using the Pico's internal pull-up resistor.

## Goal
Learn how to eliminate external pull-up/pull-down resistors by enabling internal resistor configurations on the RP2040 chip.

## What You Will Build
An active-LOW button switch circuit:
- **Push Button (GP16)**: Monitored via internal pull-up. The pin reads `HIGH` by default when the button is open, and goes `LOW` when pressed.
- **Warning LED (GP15)**: Turns ON when the button is pressed (LOW state).

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
| Push Button | Terminal 2 | GND | Connects directly to ground |
| Red LED | Anode | GP15 | Control output |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int BUTTON_PIN = 16;
const int LED_PIN    = 15;

void setup() {
  pinMode(BUTTON_PIN, INPUT_PULLUP); // Enable internal pull-up resistor
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int buttonState = digitalRead(BUTTON_PIN);

  // Active-LOW logic: button pressed pulls pin to ground (LOW)
  if (buttonState == LOW) {
    digitalWrite(LED_PIN, HIGH);
  } else {
    digitalWrite(LED_PIN, LOW);
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Push Button**, and **Red LED** onto the canvas.
2. Connect Button: **Terminal 1** to **GP16**, **Terminal 2** to **GND** (no external resistor needed!).
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Click and hold the push button to turn on the LED.

## Expected Output

Terminal:
```
Simulation active. Internal pull-up active on GP16.
```

## Expected Canvas Behavior
| Button Input (GP16) | Pin Voltage | LED state (GP15) |
| --- | --- | --- |
| open (Released) | 3.3V (HIGH) | LOW (OFF) |
| Closed (Pressed) | 0V (LOW) | HIGH (ON) |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `pinMode(16, INPUT_PULLUP)` | Connects GP16 to internal 50k-ohm pull-up resistor tied to 3.3V inside the RP2040 chip. |
| `buttonState == LOW` | Checks if the pin has been pulled to ground by the pressed switch contact. |

## Hardware & Safety Concept: Active-LOW Inputs
In active-LOW configurations, closing the switch connects the pin directly to GND, avoiding routing power rails (3.3V or 5V) to panel switches. This reduces the risk of accidental short-circuits to chassis grounds, making it the standard approach in industrial electronics.

## Try This! (Challenges)
1. **Flash Switch**: Make the LED flash rapidly when the button is pressed.
2. **Reverse Logic Indicator**: Turn the LED ON when the button is open (released).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays ON constantly | Switch wired to VCC | Check that the button connects GP16 to GND, not 3.3V. |

## Mode Notes
This basic GPIO project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [16 - Pico Button LED](16-pico-button-led.md)
- [18 - Pico Button Pull-down](18-pico-button-pulldown.md)
