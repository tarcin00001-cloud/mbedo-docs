# 63 - Pico TM1637 Counter

Display a live push-button event count on a TM1637 7-segment display.

## Goal
Learn how to combine digital inputs (push buttons) and numeric display outputs (TM1637) to build a physical counter.

## What You Will Build
A digital press counter:
- **Push Button (GP15)**: Configured with an internal pull-up (active-LOW).
- **TM1637 Display (CLK GP16, DIO GP17)**: Shows the current press count, incrementing by 1 each time the button is pressed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| TM1637 4-Digit Display | `tm1637` | Yes | Yes |
| Push Button | `button` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| TM1637 | VCC | 3.3V | Power supply |
| TM1637 | CLK | GP16 | Clock control |
| TM1637 | DIO | GP17 | Data input/output |
| TM1637 | GND | GND | Ground reference |
| Push Button | Terminal 1 | GP15 | Sense pin |
| Push Button | Terminal 2 | GND | Ground return |

## Code
```cpp
#include <TM1637Display.h>

const int BUTTON_PIN = 15;
const int CLK_PIN    = 16;
const int DIO_PIN    = 17;

TM1637Display display(CLK_PIN, DIO_PIN);

int count = 0;
bool lastButtonState = HIGH;

void setup() {
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  display.setBrightness(0x0f);
  display.showNumberDec(count); // Start display at 0
}

void loop() {
  int currentButtonState = digitalRead(BUTTON_PIN);

  // Transition check: falling edge (pressed)
  if (currentButtonState == LOW && lastButtonState == HIGH) {
    count = count + 1;
    display.showNumberDec(count);
    delay(150); // Debounce delay
  }

  lastButtonState = currentButtonState;
  delay(10);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **TM1637 Display**, and **Push Button** onto the canvas.
2. Connect TM1637: **VCC** to **3V3**, **CLK** to **GP16**, **DIO** to **GP17**, **GND** to **GND**.
3. Connect Button: **Terminal 1** to **GP15**, **Terminal 2** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Click the button multiple times and watch the display count increase.

## Expected Output

Terminal:
```
Simulation active. Counting clicks on GP15.
```

## Expected Canvas Behavior
* Start state: Display shows `0`
* 1 press: Display shows `1`
* 5 presses: Display shows `5`

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `display.showNumberDec(count)` | Formats the integer variable value and sends it over CLK/DIO lines to update the segments. |

## Hardware & Safety Concept: Input Noise Filtering
Whenever buttons are operated by users, electrical noise spikes from physical contact friction can trigger false counts. Adding a software debounce window (e.g. `delay(150)`) makes the program ignore further inputs for 150 ms after a click is detected, ensuring only true presses are registered.

## Try This! (Challenges)
1. **Clear Button**: Wire a second button to GP14 that resets the display count to 0 when clicked.
2. **Alert limit**: Sound a buzzer on GP10 if the count reaches 10.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Count increases by 2 or more on a single click | Debounce delay missing | Verify that the `delay(150)` step is placed inside the button transition check `if` statement. |

## Mode Notes
This basic counter control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [26 - Pico Button Counter](../../beginner/26-pico-button-counter.md)
- [62 - Pico TM1637 Print](62-pico-tm1637-print.md)
