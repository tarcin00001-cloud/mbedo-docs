# 17 - Hold-to-Light

Keep an LED on for one second after a button is pressed and released.

## Goal
Learn how to use the non-blocking `millis()` function to track time and keep an output active for a specific duration after an event.

## What You Will Build
Pressing the button once turns the LED on immediately. The LED stays on for exactly one second (1000 ms) after the button is pressed, then automatically turns off.

**Why D2 and D13?** Pin D2 reads the user button presses while Pin D13 drives the visual indicator LED.

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
const int LED_PIN    = 13;

unsigned long lastTime = 0; // Stores the last time the button was pressed

void setup() {
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("Hold-to-Light Ready");
}

void loop() {
  // If the button is pressed, update our timestamp
  if (digitalRead(BUTTON_PIN) == LOW) {
    lastTime = millis();
  }

  // Check if less than 1 second (1000 ms) has passed since the last press
  if (millis() - lastTime < 1000) {
    digitalWrite(LED_PIN, HIGH);
  } else {
    digitalWrite(LED_PIN, LOW);
  }
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **Push Button**, and **LED** onto the canvas.
2. Connect LED **A** (Anode) to Arduino **D13** and LED **C** (Cathode) to Arduino **GND**.
3. Connect Button **1.r** to Arduino **D2** and Button **2.r** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Click the button on the canvas once. The LED will light up and stay lit for 1 second before turning off automatically.

## Expected Output

Terminal:
```
Hold-to-Light Ready
```

### Expected Canvas Behavior

| Time Since Press | Button State | Pin D13 Output | LED State |
| --- | --- | --- | --- |
| Idle | Released (HIGH) | LOW (0V) | OFF |
| 0 - 1000 ms | Pressed (LOW) | HIGH (5V) | ON |
| 1000 ms+ | Released (HIGH) | LOW (0V) | OFF |

The LED remains ON for exactly 1 second after a button click before turning off.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `unsigned long lastTime` | Declares a variable to store the timestamp of the last button press. We use `unsigned long` because `millis()` returns a very large number that grows continuously. |
| `lastTime = millis()` | Stores the current running milliseconds count when the button reads LOW. |
| `if (millis() - lastTime < 1000)` | Calculates how much time has elapsed since the last button press. If it is less than 1000 ms, the LED stays HIGH. Once 1000 ms passes, the condition becomes false and the `else` block turns the LED LOW. |

## Hardware & Safety Concept: Blocking vs Non-Blocking Code
Using `delay(1000)` stops the microcontroller completely - it cannot read inputs or run any other code during that time. Using `millis()` is **non-blocking**; it works like looking at a wall clock to measure elapsed time, allowing the Arduino to continue checking inputs and running code in real-time.

## Try This! (Challenges)
1. **Extend the Hold Time**: Change the code so that the LED stays on for 3 seconds (3000 ms) after the press.
2. **Serial Countdown**: Print the remaining hold time in milliseconds to the Terminal while the LED is on.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays on constantly | The button pin is floating or wired incorrectly | Ensure `BUTTON_PIN` is configured as `INPUT_PULLUP` and wired to GND. |
| LED turns off instantly when button is released | Math condition mismatch | Make sure the syntax matches `millis() - lastTime < 1000` exactly. |

## Mode Notes
This specific `millis()` timing pattern is parsed and modeled by the MbedO interpreted runtime.

## Related Projects
- [11 - Button ON/OFF](11-button-onoff.md)
- [12 - Button Toggle](12-button-toggle.md)
