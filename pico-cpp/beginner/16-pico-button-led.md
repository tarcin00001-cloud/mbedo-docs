# 16 - Pico Button LED

Control an LED using a push button input.

## Goal
Learn how to configure a GPIO pin as a digital input, read its logic state, and use it to switch a digital output.

## What You Will Build
A momentary button switch circuit:
- **Push Button (GP16)**: Monitored via digital input.
- **Warning LED (GP15)**: Turns ON only while the button is pressed (HIGH state) and turns OFF when released.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 10k-ohm Resistor | `resistor` | Optional | Yes (pull-down) |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Push Button | Terminal 1 | 3.3V | Input voltage source |
| Push Button | Terminal 2 | GP16 | Sense pin (needs 10k pull-down to GND) |
| Red LED | Anode | GP15 | Control output |
| Red LED | Cathode | GND | Ground return via 220-ohm resistor |

## Code
```cpp
const int BUTTON_PIN = 16;
const int LED_PIN    = 15;

void setup() {
  pinMode(BUTTON_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int buttonState = digitalRead(BUTTON_PIN);

  if (buttonState == HIGH) {
    digitalWrite(LED_PIN, HIGH); // Button pressed, light ON
  } else {
    digitalWrite(LED_PIN, LOW);  // Button released, light OFF
  }

  delay(50); // Small debounce delay
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Push Button**, and **Red LED** onto the canvas.
2. Connect Button: **Terminal 1** to **3V3**, **Terminal 2** to **GP16** (wire a 10k resistor from GP16 to GND).
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Paste code, run the simulation, and hold click on the button.

## Expected Output

Terminal:
```
Simulation active. Reading GP16 input to drive GP15.
```

## Expected Canvas Behavior
| Button Input (GP16) | LED state (GP15) | LED Visual |
| --- | --- | --- |
| LOW (Released) | LOW | OFF (Dark) |
| HIGH (Pressed) | HIGH | ON (Red light) |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `pinMode(BUTTON_PIN, INPUT)` | Configures GP16 to read voltage levels high (3.3V) or low (0V). |
| `digitalRead(BUTTON_PIN)` | Returns the current state of GP16: `HIGH` or `LOW`. |

## Hardware & Safety Concept: Pull-Down Resistors
When the button is open (released), the input pin GP16 is disconnected from 3.3V. Without a connection, the pin "floats" and detects static electrical noise, causing the LED to turn ON and OFF randomly. A **pull-down resistor** (typically 10k ohms) ties the pin to GND, ensuring a stable `LOW` state when the button is open.

## Try This! (Challenges)
1. **Toggle Switch**: Modify the code so that pressing the button once turns the LED ON, and pressing it again turns it OFF.
2. **Inverted Logic**: Program the LED to stay ON by default and turn OFF only when the button is pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED flashes randomly when untouched | Missing pull-down resistor | Check that a 10k resistor is wired between GP16 and GND. |

## Mode Notes
This basic GPIO project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [17 - Pico Button Pull-up](17-pico-button-pullup.md)
- [19 - Pico Button AND Gate](19-pico-button-and.md)
