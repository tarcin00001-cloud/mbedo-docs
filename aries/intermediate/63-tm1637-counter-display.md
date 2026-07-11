# 63 - TM1637 Counter Display

Build an interactive push-button counter that displays the incrementing count on a TM1637 7-segment display using the VEGA ARIES v3 board.

## Goal
Learn how to detect button presses, implement software debouncing, and update a multi-digit 7-segment display dynamically in a loop-free C++ state machine.

## What You Will Build
An interactive counter system where pressing a push button connected to GPIO 16 increments a decimal counter, showing the updated count (0, 1, 2...) on the TM1637 display.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| TM1637 4-Digit Display | `tm1637` | Yes | Yes |
| Push Button | `button` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| TM1637 Display | CLK | GPIO 14 | Blue | Clock control pin |
| TM1637 Display | DIO | GPIO 15 | Yellow | Data pin |
| TM1637 Display | VCC | 3V3 | Red | Power supply |
| TM1637 Display | GND | GND | Black | Ground |
| Push Button | Pin 1 | GPIO 16 | Green | Button signal pin |
| Push Button | Pin 2 | GND | Black | Ground reference |

> **Wiring tip:** Configure the push button on GPIO 16 with `INPUT_PULLUP` in code. This pulls the signal pin HIGH until the button is pressed, which routes it to GND and sends a LOW state.

## Code
```cpp
#include <TM1637Display.h>

const int CLK_PIN = 14;
const int DIO_PIN = 15;
const int BUTTON_PIN = 16;

TM1637Display display(CLK_PIN, DIO_PIN);
int count = 0;
int lastButtonState = HIGH;

void setup() {
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  display.setBrightness(0x0f);
  display.showNumberDec(count); // Initialize the display to show 0
}

void loop() {
  int currentButtonState = digitalRead(BUTTON_PIN);

  // Detect transition from HIGH to LOW (falling edge button press)
  if (currentButtonState == LOW && lastButtonState == HIGH) {
    count = count + 1;
    display.showNumberDec(count);
    delay(200); // Software debounce delay to prevent contact bounce counts
  }

  lastButtonState = currentButtonState;
  delay(10); // Polling delay
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **TM1637 Display**, and **Push Button** components onto the canvas.
2. Wire the TM1637: **CLK** to **GPIO 14**, **DIO** to **GPIO 15**, **VCC** to **3V3**, and **GND** to **GND**.
3. Wire the Push Button: **Pin 1** to **GPIO 16**, **Pin 2** to **GND**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Button Pressed. Current Count: 1
Button Pressed. Current Count: 2
```

## Expected Canvas Behavior
* The TM1637 starts displaying `0`. Clicking the push button on the canvas increments the display digits by one.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(BUTTON_PIN, INPUT_PULLUP)` | Configures pin 16 with internal pull-up to prevent floating pin state. |
| `digitalRead(BUTTON_PIN)` | Reads pin logic value (LOW indicates button is pressed). |
| `count = count + 1` | Increments the counter state variable. |
| `display.showNumberDec(count)` | Pushes the updated count value to the TM1637 display segments. |
| `delay(200)` | Software debounce to ignore mechanical contact bounce. |

## Hardware & Safety Concept
* **Switch Debouncing**: When mechanical switches are pressed, their metal contacts bounce against each other, making and breaking contact for a few milliseconds. Microcontrollers are fast enough to detect this as multiple presses. Software debouncing (like a 200 ms delay or time comparison) ignores these fast transitions.
* **Internal Pull-Ups**: Internal pull-up resistors are embedded within the SoC's GPIO blocks. They default the pin state to 3.3V (HIGH) so that an external switch only needs to connect the pin to GND (LOW) to register a state transition, eliminating external resistors.

## Try This! (Challenges)
1. **Count Down**: Add logic so that the count decreases by 1 on button press if the onboard warning LED on GPIO 15 is active.
2. **Auto Reset**: Modify the logic to reset the count back to 0 automatically if the counter exceeds 99.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Pressing button increments count by multiple values | Debounce delay missing | Ensure `delay(200)` is included inside the button-press detection block. |
| Display increments without pressing button | Floating input pin | Make sure the pin mode is set to `INPUT_PULLUP` instead of `INPUT`. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [62 - TM1637 7-Segment Display Print Numbers](62-tm1637-7-segment-display-print-numbers.md)
- [68 - Stepper Motor Position Log (LCD display)](68-stepper-motor-position-log.md)
