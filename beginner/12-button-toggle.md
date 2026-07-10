# 12 - Button Toggle

Toggle an LED on and off with a single button press.

## Goal
Learn how to use state tracking variables to save the status of an output, allowing a single button click to toggle between ON and OFF states instead of requiring the button to be held down.

## What You Will Build
Pressing the button once turns the LED on. The LED stays on after you release the button. Pressing the button a second time turns the LED off.

**Why D2 and D13?** Pin D2 handles input tracking while pin D13 drives the output state visually on both the canvas and the onboard LED.

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

int ledState = LOW;        // Track the current state of the LED (HIGH or LOW)
int lastButtonState = HIGH; // Track the previous state of the button

void setup() {
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, ledState);
  
  Serial.begin(9600);
  Serial.println("Button Toggle Ready");
}

void loop() {
  int buttonState = digitalRead(BUTTON_PIN);

  // Check if the button transition changed from HIGH to LOW (just pressed)
  if (buttonState == LOW && lastButtonState == HIGH) {
    // Toggle the LED state variable
    if (ledState == LOW) {
      ledState = HIGH;
      Serial.println("Toggle - ON");
    } else {
      ledState = LOW;
      Serial.println("Toggle - OFF");
    }
    digitalWrite(LED_PIN, ledState);
    
    // Simple software debounce delay to filter mechanical noise
    delay(150);
  }
  
  // Save the current state for the next comparison
  lastButtonState = buttonState;
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **Push Button**, and **LED** onto the canvas.
2. Connect LED **A** (Anode) to Arduino **D13** and LED **C** (Cathode) to Arduino **GND**.
3. Connect Button **1.r** to Arduino **D2** and Button **2.r** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Click the button on the canvas once to turn the LED on, and click it again to turn it off.

## Expected Output

Serial Monitor:
```
Button Toggle Ready
Toggle - ON
Toggle - OFF
...
```

### Expected Canvas Behavior

| Button Event | lastButtonState | buttonState | ledState Result | LED Pin D13 |
| --- | --- | --- | --- | --- |
| Default | HIGH | HIGH | LOW | LOW (0V) - OFF |
| Click Starts | HIGH | LOW | HIGH | HIGH (5V) - ON |
| Click Held | LOW | LOW | HIGH (no change) | HIGH (5V) - ON |
| Click Released | LOW | HIGH | HIGH (no change) | HIGH (5V) - ON |
| Click 2 Starts | HIGH | LOW | LOW | LOW (0V) - OFF |

The LED state changes only on the transition edge of a button press.

## Code Walkthrough

| Line / Variable | What It Does |
| --- | --- |
| `int ledState = LOW;` | Declares a state variable. We write this variable to the LED pin to remember the ON/OFF state between loops. |
| `lastButtonState` | Remembers what the button was reading in the previous loop execution, allowing edge-detection. |
| `if (buttonState == LOW && lastButtonState == HIGH)` | This is only true during the exact transition from unpressed (HIGH) to pressed (LOW). It prevents the code inside from looping continuously while the button is held down. |

## Hardware & Safety Concept: Contact Bounce
Mechanical buttons contain small metal leaves. When pressed, these contacts strike each other and bounce slightly for a few milliseconds before forming a stable contact. Microcontrollers are so fast that they detect these bounces as multiple rapid presses, causing the LED to toggle on and off randomly. A **debounce delay** (`delay(150)`) bypasses this noise by ignoring any inputs immediately following the first transition.

## Try This! (Challenges)
1. **Reduce Debounce**: Change `delay(150)` to `delay(5)`. Press the button repeatedly. Does it still toggle cleanly every time?
2. **Beep on Toggle**: Wire a Buzzer to D8 and generate a short chirp (`tone(8, 2000, 50);`) only when the LED toggles ON.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED toggles rapidly when the button is held | Missing lastButtonState check | Ensure you check both `buttonState == LOW` and `lastButtonState == HIGH` in the conditional test. |
| LED does not change state | Debounce time too high or bad wiring | Verify D2 is connected to the button and the code has `delay(150)`. |

## Mode Notes
These patterns (state variables, transition logic, and simple conditional branches) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [11 - Button ON/OFF](11-button-onoff.md)
- [13 - Button Counter Print](13-button-counter-print.md)
