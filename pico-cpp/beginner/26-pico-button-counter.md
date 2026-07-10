# 26 - Pico Button Counter

Count and log push button presses to the Serial Monitor.

## Goal
Learn how to use state-change detection (detecting transitions rather than steady states) to accurately count user input events.

## What You Will Build
An event counter:
- **Push Button (GP16)**: Configured with internal pull-up (active-LOW).
- **Serial Monitor**: Increments and prints the click count ("Press Count: X") each time the button is pressed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Push Button | `button` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Push Button | Terminal 1 | GP16 | Digital sense pin |
| Push Button | Terminal 2 | GND | Ground return |

## Code
```cpp
const int BUTTON_PIN = 16;

int pressCount = 0;
bool lastButtonState = HIGH;

void setup() {
  Serial.begin(9600);
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  Serial.println("Press Counter Active");
}

void loop() {
  int currentButtonState = digitalRead(BUTTON_PIN);

  // Transition check: was HIGH (released) and is now LOW (pressed)
  if (currentButtonState == LOW && lastButtonState == HIGH) {
    pressCount = pressCount + 1; // Increment counter
    Serial.print("Press Count: ");
    Serial.println(pressCount);
    delay(150); // Debounce delay
  }

  lastButtonState = currentButtonState;
  delay(10);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Push Button** onto the canvas.
2. Connect Button: **Terminal 1** to **GP16**, **Terminal 2** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Press the button multiple times and look at the Serial console logs.

## Expected Output

Terminal:
```
Press Counter Active
Press Count: 1
Press Count: 2
Press Count: 3
```

## Expected Canvas Behavior
* Click once: Terminal logs `Press Count: 1`
* Click twice: Terminal logs `Press Count: 2`

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `pressCount = pressCount + 1` | Increments the integer variable value by 1. |
| `lastButtonState = currentButtonState` | Updates the state memory for comparison in the next loop pass. |

## Hardware & Safety Concept: Rising vs. Falling Edges
- **Rising Edge**: Voltage transitions from LOW to HIGH (0V to 3.3V).
- **Falling Edge**: Voltage transitions from HIGH to LOW (3.3V to 0V).
By trigger-checking the falling edge (HIGH to LOW) on a pull-up button input, we guarantee the count increases *once* at the initial click, regardless of how long the user holds the button down.

## Try This! (Challenges)
1. **Target Alert**: Flash an LED on GP15 when `pressCount` reaches a multiple of 5 (e.g. `pressCount % 5 == 0`).
2. **Reset Trigger**: Connect a second button to GP17 that resets `pressCount` back to 0.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Press count jumps by 2 or 3 on a single press | Debounce delay too short | Increase the debounce delay in the if block from `150` to `200` ms. |

## Mode Notes
This basic state monitoring project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [16 - Pico Button LED](16-pico-button-led.md)
- [27 - Pico Double Click](27-pico-double-click.md)
