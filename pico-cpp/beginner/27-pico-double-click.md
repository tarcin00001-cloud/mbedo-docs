# 27 - Pico Double Click

Detect double-click button inputs to trigger separate control states.

## Goal
Learn how to use timing thresholds (`millis()`) in digital logic to distinguish between single-press and double-click actions.

## What You Will Build
A double-click command module:
- **Push Button (GP16)**: Reads active-LOW.
- **Warning LED (GP15)**: Turns ON (or toggles) only when a double-click is registered.
- **Serial Monitor**: Prints "Double Click Detected!" when two clicks happen within 400 ms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Push Button | Terminal 1 | GP16 | Sense pin |
| Push Button | Terminal 2 | GND | Ground return |
| Red LED | Anode | GP15 | Double-click status |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int BUTTON_PIN = 16;
const int LED_PIN    = 15;

unsigned long lastClickTime = 0;
bool lastButtonState = HIGH;
bool ledState = false;

void setup() {
  Serial.begin(9600);
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  Serial.println("Double Click Monitor Active");
}

void loop() {
  int currentButtonState = digitalRead(BUTTON_PIN);
  unsigned long currentTime = millis();

  // Transition check: falling edge (pressed)
  if (currentButtonState == LOW && lastButtonState == HIGH) {
    // Check time gap between this click and the last click
    if (currentTime - lastClickTime < 400 && currentTime - lastClickTime > 50) {
      Serial.println("Double Click Detected!");
      ledState = !ledState;
      digitalWrite(LED_PIN, ledState ? HIGH : LOW);
    }
    lastClickTime = currentTime;
  }

  lastButtonState = currentButtonState;
  delay(10);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Push Button**, and **Red LED** onto the canvas.
2. Connect Button: **Terminal 1** to **GP16**, **Terminal 2** to **GND**.
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Click the push button twice rapidly (within 0.4 seconds) to toggle the LED.

## Expected Output

Terminal:
```
Double Click Monitor Active
Double Click Detected!
```

## Expected Canvas Behavior
* Single click: No LED change.
* Two fast clicks: LED turns ON (Red).
* Two fast clicks again: LED turns OFF.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `millis()` | Returns the number of milliseconds since the Raspberry Pi Pico board began running the current program. |
| `currentTime - lastClickTime < 400` | Calculates if the elapsed time since the previous click is less than the 400 ms double-click threshold. |

## Hardware & Safety Concept: Non-Blocking Timing
Using `millis()` instead of `delay()` is fundamental to responsive firmware design. While `delay()` pauses the entire CPU, checking elapsed milliseconds with `millis()` lets the Pico run checks (like checking other buttons, reading analog sensors, or blinking lights) continuously while waiting for timing intervals to expire.

## Try This! (Challenges)
1. **Triple Click Alert**: Modify the logic to detect a triple-click (three clicks within 700 ms) and sound a buzzer on GP14.
2. **Speed Dial**: Make the double-click threshold shorter (e.g. 250 ms) and observe how much faster you have to click.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Single press triggers a double-click | Bouncing contacts | Increase the lower boundary time limit (`currentTime - lastClickTime > 50`) to filter contact bounce. |

## Mode Notes
This basic timing logic project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [16 - Pico Button LED](16-pico-button-led.md)
- [26 - Pico Button Counter](26-pico-button-counter.md)
