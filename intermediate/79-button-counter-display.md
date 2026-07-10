# 79 - Button Counter Display

Count the number of button presses and display the total count on a 4-digit TM1637 7-segment display.

## Goal
Learn how to use digital state-change edge detection (detecting when a button goes from released to pressed) to update a running counter on a TM1637 screen.

## What You Will Build
Each time you press the button connected to pin D4, the count increases by 1. The display shows the updated count (e.g. `1`, `2`, `3`, etc.) instantly, with a brief debounce pause to prevent false counts.

**Why D2, D3, and D4?** Pins D2/D3 manage the display. Pin D4 reads the push button with an internal pull-up.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| TM1637 Display | `seven_segment_tm1637` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Push Button | Pin 1 | D4 | Input signal pin |
| Push Button | Pin 2 | GND | Ground reference |
| TM1637 Display | CLK | D2 | Serial Clock line |
| TM1637 Display | DIO | D3 | Serial Data line |
| TM1637 Display | VCC | 5V | Power supply (5V) |
| TM1637 Display | GND | GND | Ground reference |

## Code
```cpp
#include <TM1637Display.h>

const int CLK_PIN = 2;
const int DIO_PIN = 3;
const int BTN_PIN = 4;

TM1637Display display(CLK_PIN, DIO_PIN);

int counter = 0;
int lastBtnState = HIGH; // Start HIGH because of INPUT_PULLUP

void setup() {
  pinMode(BTN_PIN, INPUT_PULLUP);
  
  display.setBrightness(5);
  display.clear();
  display.showNumberDec(counter); // Display starting 0
  
  Serial.begin(9600);
  Serial.println("Push Button Counter Ready");
}

void loop() {
  int btnState = digitalRead(BTN_PIN);
  
  // Detect state change: was released (HIGH) and is now pressed (LOW)
  if (btnState == LOW && lastBtnState == HIGH) {
    counter++;
    Serial.print("Press registered. Count: ");
    Serial.println(counter);
    
    // Update display value
    display.showNumberDec(counter);
    
    // Debounce delay to prevent double-counting contact bounce
    delay(150); 
  }
  
  // Save current button state for comparison on next loop iteration
  lastBtnState = btnState;
  
  delay(10); // Small loop tick delay
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **Push Button**, and **TM1637 7-Segment Display** onto the canvas.
2. Connect Button: **pin 1** to Arduino **D4** and **pin 2** to Arduino **GND**.
3. Connect TM1637: **CLK** to Arduino **D2**, **DIO** to Arduino **D3**, **VCC** to Arduino **5V**, and **GND** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Click the Push Button on the canvas several times, and watch the counter increment on the TM1637 display.

## Expected Output
The TM1637 display component on the canvas lights up and displays:
```
   1
```
Each click on the button increments the number shown on the screen.

## Expected Canvas Behavior

| Button Press Action | State comparison | Counter Variable | TM1637 Display Output |
| --- | --- | --- | --- |
| None (Released) | HIGH == HIGH | No change | Keeps previous count |
| Press (First click) | LOW != HIGH | Increments to 1 | `   1` |
| Press (Second click) | LOW != HIGH | Increments to 2 | `   2` |

The count increments by exactly 1 per click.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `btnState == LOW && lastBtnState == HIGH` | State-change check (falling edge). Ensures the counter increments only once when the button is pushed down, rather than continuously while it is held down. |
| `lastBtnState = btnState` | Stores the button state for the next check cycle. |

## Hardware & Safety Concept: Contact Debouncing
Mechanical switch buttons consist of two springy metal plates. When you push the button, the plates do not touch cleanly; instead, they bounce and rub against each other for a fraction of a millisecond.
- To a fast microcontroller, this bounce looks like the button was pressed and released 10 to 50 times in rapid succession, resulting in a wildly incorrect count.
- In software, we filter this out using a **debounce delay** (e.g. `delay(150)`), which ignores any voltage changes for a brief period after a press is registered, allowing the metal plates to settle.

## Try This! (Challenges)
1. **Reset Button**: Add a second button on D5. If pressed, it resets the counter back to `0`.
2. **Reverse Count**: Change the logic so that the counter starts at `99` and counts down by 1 on each press.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Pressing the button once increases count by 2 or more | Debounce delay is too short or missing | Verify that `delay(150)` is included inside the `if` statement block. |
| Count increases constantly without pressing button | Pin configured as INPUT instead of INPUT_PULLUP | Ensure pin mode is set to `INPUT_PULLUP` to prevent noise from floating inputs. |

## Mode Notes
These patterns (digital edge detection driving TM1637 display updates) are supported by MbedO interpreted mode.

## Related Projects
- [02 - Button Press](../beginner/02-button-press.md)
- [06 - Button Edge Detection](../beginner/06-button-edge-detection.md)
- [78 - Potentiometer Number](78-potentiometer-number.md)
