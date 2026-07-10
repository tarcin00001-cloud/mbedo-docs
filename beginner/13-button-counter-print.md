# 13 - Button Counter Print

Count how many times a button is pressed and display the count in the Terminal.

## Goal
Learn how to increment a counter variable based on button state transitions and print variable values dynamically to the Terminal.

## What You Will Build
Each time you press the push button, the counter increments by one, and a message displaying the total count is printed to the Terminal.

**Why D2?** Pin D2 is configured as our digital input pin with pull-up logic to detect button clicks reliably.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Push Button | `button` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Push Button | 1.r | D2 | Input signal connection |
| Push Button | 2.r | GND | Ground reference |

## Code
```cpp
const int BUTTON_PIN = 2;

int buttonPushCounter = 0;   // Stores the total number of button presses
int lastButtonState = HIGH;  // Track the previous state of the button

void setup() {
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  Serial.begin(9600);
  Serial.println("Button Counter Ready");
}

void loop() {
  int buttonState = digitalRead(BUTTON_PIN);

  // Check if the button transition changed from HIGH to LOW (just pressed)
  if (buttonState == LOW && lastButtonState == HIGH) {
    buttonPushCounter = buttonPushCounter + 1;
    
    Serial.print("Number of button pushes: ");
    Serial.println(buttonPushCounter);
    
    delay(150); // Software debounce delay
  }
  
  lastButtonState = buttonState;
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** and **Push Button** onto the canvas.
2. Connect Button **1.r** to Arduino **D2** and Button **2.r** to Arduino **GND**.
3. Paste the code into the editor.
4. Use the default Arduino interpreted runtime.
5. Click **Run**.
6. Open the **Terminal** tab in the bottom dock.
7. Click the button on the canvas multiple times to see the count increase.

## Expected Output

Terminal:
```
Button Counter Ready
Number of button pushes: 1
Number of button pushes: 2
...
```

### Expected Canvas Behavior

| Action | lastButtonState | buttonState | counter value | Terminal Printout |
| --- | --- | --- | --- | --- |
| Start | HIGH | HIGH | 0 | No new print |
| 1st Click | HIGH | LOW | 1 | "Number of button pushes: 1" |
| Release | LOW | HIGH | 1 | No new print |
| 2nd Click | HIGH | LOW | 2 | "Number of button pushes: 2" |

The count increments exactly once per click.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `buttonPushCounter = buttonPushCounter + 1;` | Increments the counter variable by 1. Can also be written as `buttonPushCounter++;`. |
| `Serial.print("...")` | Prints the text message inside quotes without moving to a new line. |
| `Serial.println(buttonPushCounter)` | Prints the value of the counter and adds a new line character, forcing the next print statement to start below. |

## Hardware & Safety Concept: Digital Logic Levels
A digital input pin only resolves two discrete states: `HIGH` (representing 5V) or `LOW` (representing 0V). When using `INPUT_PULLUP`, the pin reads `HIGH` (logic level 1) when the switch is open. Pressing the button shorts the pin directly to Ground, forcing it to read `LOW` (logic level 0).

## Try This! (Challenges)
1. **Reset Button**: Wire a second button to D3 and modify the code so that pressing this second button resets the counter back to 0 and prints "Counter Reset".
2. **Milestone Alerts**: Add an `if` statement to check if the count is a multiple of 5 (using the modulo operator: `if (buttonPushCounter % 5 == 0)`) and print a special milestone message.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Counter jumps up by multiple numbers on a single press | Debounce delay missing | Make sure `delay(150)` is inside the conditional block. |
| Counter prints continuously without pressing the button | Button wired incorrectly or missing `INPUT_PULLUP` | Verify wiring (D2 and GND) and check that `INPUT_PULLUP` is declared in `setup()`. |

## Mode Notes
These patterns (integer arithmetic, edge detection, and print statements) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [12 - Button Toggle](12-button-toggle.md)
- [15 - Analog Meter Serial](15-analog-meter-serial.md)
