# 26 - ESP32 Button Press Counter

Count individual button presses and print the running total to the Serial Monitor using rising-edge detection and software debouncing.

## Goal
Learn how to count discrete events from a mechanical button using edge detection, and understand why a simple level-reading loop over-counts without debounce protection.

## What You Will Build
A single push button on GPIO 4 with a 10 kΩ pull-down resistor. Each time the button is pressed (rising edge detected), a counter increments and the total is printed to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button | Pin 1 (supply side) | 3V3 | Red | Supply voltage |
| Push Button | Pin 2 (signal side) | GPIO4 | Yellow | Counter input pin |
| 10 kΩ Resistor | Leg 1 | GPIO4 | White | Pull-down: holds pin LOW when button released |
| 10 kΩ Resistor | Leg 2 | GND | Black | Ground reference |

> **Wiring tip:** The 10 kΩ pull-down is essential for accurate counting. Without it the pin floats HIGH unpredictably, causing phantom counts. No LED is required — this project is pure input counting and serial logging.

## Code
```cpp
// Button Press Counter with edge detection and debounce
const int BTN_PIN = 4;

int  pressCount  = 0;      // Running total
bool lastState   = false;  // Previous button reading
unsigned long lastDebounce = 0;
const unsigned long DEBOUNCE_MS = 50;

void setup() {
  pinMode(BTN_PIN, INPUT);   // External pull-down fitted
  Serial.begin(115200);
  Serial.println("Button Press Counter ready. Press the button!");
}

void loop() {
  bool currentState = digitalRead(BTN_PIN);
  unsigned long now = millis();

  // Detect a rising edge with debounce
  if (currentState && !lastState && (now - lastDebounce > DEBOUNCE_MS)) {
    pressCount++;
    lastDebounce = now;
    Serial.print("Button pressed! Total count: ");
    Serial.println(pressCount);
  }

  lastState = currentState;
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Button** onto the canvas.
2. Connect Button **output** to **GPIO4**.
3. Paste the code and click **Run**.
4. Click the button widget repeatedly — watch the counter increment in the Serial Monitor.

## Expected Output
Serial Monitor:
```
Button Press Counter ready. Press the button!
Button pressed! Total count: 1
Button pressed! Total count: 2
Button pressed! Total count: 3
Button pressed! Total count: 4
```

## Expected Canvas Behavior
* No canvas component changes visually — output is entirely on the Serial Monitor.
* Each single click of the button widget increments the count by exactly 1.
* Holding the button widget does not increment the counter repeatedly.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `bool lastState = false` | Stores the previous button reading to allow edge comparison. |
| `unsigned long lastDebounce = 0` | Stores the timestamp of the last confirmed press for debounce timing. |
| `if (currentState && !lastState && (now - lastDebounce > DEBOUNCE_MS))` | Three-part guard: (1) button is HIGH now, (2) it was LOW last loop (rising edge), (3) enough time has passed since the last press. |
| `pressCount++` | Increments the integer counter by one. |
| `lastDebounce = now` | Records the time of this valid press to start the debounce window. |
| `lastState = currentState` | Updates the edge-detection memory at the end of every loop. |

## Hardware & Safety Concept: Mechanical Bounce and Debouncing
When a mechanical button contact closes, the metal leaves flex back and forth several times before settling — a phenomenon called **contact bounce**. Each bounce registers as a separate LOW→HIGH→LOW transition. Without debounce logic, a single press can generate 5–50 false counts within the first 10–50 ms. Debouncing solves this by imposing a **lockout window** after each valid edge: any further transitions within that window are ignored. The `millis()`-based approach used here is **non-blocking** — the loop continues running during the debounce window rather than sitting in a `delay()` freeze, which makes the firmware more responsive and accurate.

## Try This! (Challenges)
1. **Reset on 10**: Reset the counter to zero automatically after every 10 presses.
2. **Double-press detection**: Detect when two presses occur within 400 ms and print "DOUBLE PRESS".
3. **LED progress bar**: Use 4 LEDs on GPIO 12–15 and light one more for every 25 presses (0–25–50–75–100).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Counter jumps by 3–10 per press | Missing or too-short debounce | Increase `DEBOUNCE_MS` from 50 to 100 |
| Counter never increments | Pull-down missing — pin reads HIGH constantly | Add 10 kΩ between GPIO 4 and GND; verify rising edge fires |
| Counter increments while holding | Level-read instead of edge-detection | Confirm `lastState` is correctly updated at the loop end |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [16 - ESP32 Button-Controlled LED](16-esp32-button-controlled-led.md)
- [27 - ESP32 Double-Click Event Filter](27-esp32-double-click-event-filter.md)
- [22 - ESP32 Touch Toggle Controller](22-esp32-touch-toggle-controller.md)
