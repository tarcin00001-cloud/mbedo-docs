# 11 - Button ON/OFF

Turn an LED on and off by pressing a push button.

## Goal
Learn how to read a digital input (a button) and use it to control a digital output (an LED) with conditional statements.

## What You Will Build
Pressing and holding the button turns the LED on. Releasing the button turns the LED off.

**Why D2 and D13?** Pin D2 supports hardware interrupts on the Arduino Uno, which makes it a preferred choice for push button inputs. Pin D13 is used because it maps directly to the Uno's onboard LED, allowing quick verification.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Push Button | 1.r | D2 | Input signal connection |
| Push Button | 2.r | GND | Ground reference |
| LED | A (Anode, longer leg) | D13 | Output signal connection |
| LED | C (Cathode, shorter leg) | GND | Ground reference |

## Code
```cpp
const int BUTTON_PIN = 2;
const int LED_PIN = 13;

void setup() {
  // Use INPUT_PULLUP to use the Arduino's internal pull-up resistor
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
  
  Serial.begin(9600);
  Serial.println("Button ON/OFF Ready");
}

void loop() {
  // Read the state of the button
  int buttonState = digitalRead(BUTTON_PIN);
  
  // Under INPUT_PULLUP, LOW means the button is pressed (connected to GND)
  if (buttonState == LOW) {
    digitalWrite(LED_PIN, HIGH);
    Serial.println("Button Pressed - LED ON");
  } else {
    digitalWrite(LED_PIN, LOW);
  }
  delay(50); // Small delay to debounce the signal
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **Push Button**, and **LED** onto the canvas.
2. Connect LED **A** (Anode) to Arduino **D13** and LED **C** (Cathode) to Arduino **GND**.
3. Connect Button **1.r** to Arduino **D2** and Button **2.r** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Click and hold the button on the canvas to see the LED light up.

## Expected Output

Serial Monitor:
```
Button ON/OFF Ready
Button Pressed - LED ON
Button Pressed - LED ON
...
```

### Expected Canvas Behavior

| Button Action | Pin D2 Reading | Pin D13 Output | LED State |
| --- | --- | --- | --- |
| Released | HIGH (5V) | LOW (0V) | OFF |
| Pressed | LOW (0V) | HIGH (5V) | ON |

The LED on the canvas lights up only when the button is held down.

## Code Walkthrough

| Line / Block | What It Does |
| --- | --- |
| `pinMode(BUTTON_PIN, INPUT_PULLUP)` | Configures pin D2 as an input and enables a built-in pull-up resistor. This keeps the pin state at `HIGH` (5V) until the button is pressed, pulling it to `LOW` (GND). |
| `digitalRead(BUTTON_PIN)` | Reads the current voltage level of pin D2, returning either `HIGH` or `LOW`. |
| `if (buttonState == LOW)` | Checks if the button is pressed. Because we are using pull-up logic, a pressed button connects the pin to GND (LOW). |

## Hardware & Safety Concept: Pull-up Resistors
If you configure a pin as a simple `INPUT` without a resistor, the pin is "floating". It behaves like an antenna, picking up stray electromagnetic noise and randomly switching between `HIGH` and `LOW`. A **pull-up resistor** connects the pin to 5V to ensure it remains stable at `HIGH` when the button is open.

## Try This! (Challenges)
1. **Reverse Logic**: Modify the code so that the LED is normally ON, and turns OFF only when you press the button.
2. **Add Release Message**: Print "Button Released" to the Terminal exactly once when the button is let go.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays on constantly | Button wired to VCC/5V instead of GND | Connect button pin 2.r to GND. |
| Button doesn't turn on LED | Missing `INPUT_PULLUP` configuration | Check setup() to make sure you wrote `INPUT_PULLUP` instead of `INPUT`. |
| Terminal shows garbled text | Baud rate mismatch | Ensure the Terminal baud rate is set to 9600. |

## Mode Notes
These patterns (`pinMode`, `digitalRead`, `digitalWrite`, conditional tests) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [01 - LED Blink](01-led-blink.md)
- [12 - Button Toggle](12-button-toggle.md)
