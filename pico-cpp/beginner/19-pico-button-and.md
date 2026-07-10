# 19 - Pico Button AND Gate

Create a logical AND gate circuit using two push buttons and a warning LED.

## Goal
Learn how to combine multiple digital inputs using logical operators (`&&`) to implement control logic.

## What You Will Build
A dual-safety trigger console:
- **Button 1 (GP16)** & **Button 2 (GP17)**: Configured with internal pull-up resistors (active-LOW).
- **Warning LED (GP15)**: Turns ON only when **both** buttons are pressed simultaneously.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Push Buttons | `button` | Yes (two buttons) | Yes |
| LED (Red) | `led` | Yes | Yes |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Button 1 | Terminal 1 | GP16 | Sense pin 1 |
| Button 1 | Terminal 2 | GND | Ground return |
| Button 2 | Terminal 1 | GP17 | Sense pin 2 |
| Button 2 | Terminal 2 | GND | Ground return |
| Red LED | Anode | GP15 | Control output |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int BUTTON1 = 16;
const int BUTTON2 = 17;
const int LED_PIN = 15;

void setup() {
  pinMode(BUTTON1, INPUT_PULLUP);
  pinMode(BUTTON2, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  // Active-LOW inputs: LOW means pressed
  int state1 = digitalRead(BUTTON1);
  int state2 = digitalRead(BUTTON2);

  // Logical AND check
  if (state1 == LOW && state2 == LOW) {
    digitalWrite(LED_PIN, HIGH); // Both pressed, light ON
  } else {
    digitalWrite(LED_PIN, LOW);  // Otherwise, light OFF
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two Push Buttons**, and **Red LED** onto the canvas.
2. Connect Button 1: **Terminal 1** to **GP16**, **Terminal 2** to **GND**.
3. Connect Button 2: **Terminal 1** to **GP17**, **Terminal 2** to **GND**.
4. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
5. Paste code, select the interpreted mode, and click **Run**.
6. Hold shift and click both buttons to turn on the LED.

## Expected Output

Terminal:
```
Simulation active. GP16 and GP17 AND verification logic active.
```

## Expected Canvas Behavior
| Button 1 (GP16) | Button 2 (GP17) | LED state (GP15) |
| --- | --- | --- |
| open (Released) | open (Released) | LOW (OFF) |
| Closed (Pressed)| open (Released) | LOW (OFF) |
| open (Released) | Closed (Pressed)| LOW (OFF) |
| Closed (Pressed)| Closed (Pressed)| HIGH (ON) |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `state1 == LOW && state2 == LOW` | Evaluates true only if both input conditions are pulled to ground. |

## Hardware & Safety Concept: Two-Hand Controls
Two-hand control logic is used in dangerous industrial machinery (like stamping presses or paper cutters) to ensure that the operator's hands are away from the danger zone. The machine only cycles when two physically separated buttons are pressed at the same time.

## Try This! (Challenges)
1. **Adding Audio**: Trigger a buzzer on GP14 to sound if only one button is pressed, indicating an incomplete safety sequence.
2. **Reverse Safety Check**: Turn the LED ON by default, and turn it OFF only when both buttons are pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED turns ON when only one button is pressed | Logical error in code | Confirm the use of the `&&` (AND) operator instead of `||` (OR). |

## Mode Notes
This basic logical operation project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [16 - Pico Button LED](16-pico-button-led.md)
- [20 - Pico Button OR Gate](20-pico-button-or.md)
