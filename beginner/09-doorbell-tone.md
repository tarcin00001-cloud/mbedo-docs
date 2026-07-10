# 09 - Doorbell Tone

Play a two-tone "ding-dong" doorbell sound when a button is pressed.

## Goal
Learn how to use a digital input (push button) to trigger a timed sequence of acoustic tones on a buzzer.

## What You Will Build
Pressing the push button connected to pin D2 triggers a classic two-note doorbell sequence (E5 at 659 Hz, then C5 at 523 Hz) on a buzzer connected to pin D8.

**Why D2 and D8?** D2 supports hardware interrupts (useful for buttons in advanced sketches), and D8 handles the audio output signal cleanly.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| 100 ohm resistor | `resistor` | Optional | Yes (optional, protects buzzer) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Push Button | 1.r | D2 | Input signal pin |
| Push Button | 2.r | GND | Ground connection |
| Buzzer | Pin 1 (Positive) | D8 | Audio control line |
| Buzzer | Pin 2 (Negative) | GND | Ground connection |

## Code
```cpp
const int BTN_PIN  = 2;
const int BUZZ_PIN = 8;

void setup() {
  // Use INPUT_PULLUP to keep the input HIGH when button is unpressed
  pinMode(BTN_PIN, INPUT_PULLUP);
  
  Serial.begin(9600);
  Serial.println("Doorbell ready");
}

void loop() {
  // Read the button state. Under INPUT_PULLUP, LOW means pressed.
  if (digitalRead(BTN_PIN) == LOW) {
    Serial.println("Ding dong!");
    
    // Play "Ding" (Note E5: 659 Hz)
    tone(BUZZ_PIN, 659); 
    delay(400); 
    noTone(BUZZ_PIN); 
    delay(100); // Brief pause
    
    // Play "Dong" (Note C5: 523 Hz)
    tone(BUZZ_PIN, 523); 
    delay(600); 
    noTone(BUZZ_PIN);
    
    // Delay to prevent re-triggering immediately
    delay(500);
  }
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **Push Button**, and **Buzzer** onto the canvas.
2. Connect Button **1.r** to Arduino **D2** and **2.r** to Arduino **GND**.
3. Connect Buzzer **pin 1** to Arduino **D8** and **pin 2** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Click the button on the canvas once to trigger the doorbell sound.

## Expected Output

Serial Monitor when button is pressed:
```
Doorbell ready
Ding dong!
```

### Expected Canvas Behavior

| Button Action | Pin D2 Reading | Buzzer Sequence | Active Pitch |
| --- | --- | --- | --- |
| Not pressed | HIGH | Silent | None |
| Pressed (Click) | LOW | Starts doorbell | E5 (659 Hz) -> C5 (523 Hz) |
| Held down | LOW | Loops sequence | Cycles with 500 ms gap |

The canvas buzzer visualizes the sound waves when the button is active.

## Code Walkthrough

| Line | What It Does |
| --- | --- |
| `pinMode(BTN_PIN, INPUT_PULLUP)` | Configures pin D2 as an input with the internal pull-up resistor active, avoiding a floating pin state. |
| `if (digitalRead(BTN_PIN) == LOW)` | Triggers the sound sequence only when the pin is pulled to Ground (button pressed). |
| `tone(BUZZ_PIN, 659)` | Plays the first note (Ding) at 659 Hz. |
| `noTone(BUZZ_PIN)` | Stops the wave generator between notes. |

## Hardware & Safety Concept: Pull-Up Logic
When a pin is configured with `INPUT_PULLUP`, the microcontroller connects it to 5V through an internal resistor. This defaults the pin to `HIGH`. When the button is pressed, it forms a direct path to Ground (`GND`), pulling the voltage down to 0V (`LOW`). This is called **negative logic** (LOW = active).

## Try This! (Challenges)
1. **Three-Tone Doorbell**: Add a third note to create a "ding-dong-ding" sequence. (Hint: Add note G5 at 784 Hz at the end).
2. **Double Doorbell Lockout**: Increase the delay at the end of the loop to `delay(3000)` to prevent people from ringing the doorbell repeatedly.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Doorbell plays constantly without stopping | Button pin is floating | Ensure `INPUT_PULLUP` is declared in `setup()`. Check that Button Pin 2.r connects to GND. |
| No sound plays when button is clicked | Ground connection missing | Verify Button 2.r is connected to GND and Buzzer Pin 2 is connected to GND. |
| Sound pitch is wrong | Frequencies incorrect | Verify the frequency arguments match E5 (659) and C5 (523). |

## Mode Notes
These patterns (`digitalRead` conditional checking triggering sequential `tone` calls) are supported by MbedO interpreted mode.

## Related Projects
- [07 - Buzzer Beep](07-buzzer-beep.md)
- [08 - Buzzer Melody](08-buzzer-melody.md)
- [11 - Button ON/OFF](11-button-onoff.md)
