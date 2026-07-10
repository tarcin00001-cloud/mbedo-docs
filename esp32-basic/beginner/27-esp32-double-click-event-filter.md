# 27 - ESP32 Double-Click Event Filter

Detect a **double-click** on a single push button — two presses within a short time window — and ignore single presses.

## Goal
Learn how to implement a time-window event filter that distinguishes between single and double clicks using `millis()` timing, a foundational pattern for gesture-based input.

## What You Will Build
A push button on GPIO 4 with a 10 kΩ pull-down resistor. The firmware watches for two rising edges within 400 ms of each other and prints "DOUBLE CLICK" when detected while treating single presses as "SINGLE CLICK".

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| LED (any colour) | `led` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button | Pin 1 (supply side) | 3V3 | Red | Supply voltage |
| Push Button | Pin 2 (signal side) | GPIO4 | Yellow | Click input |
| 10 kΩ Resistor | Leg 1 | GPIO4 | White | Pull-down: holds LOW at rest |
| 10 kΩ Resistor | Leg 2 | GND | Black | Ground reference |
| LED | Anode (+) | GPIO5 via 330 Ω | Orange | Double-click indicator |
| LED | Cathode (−) | GND | Black | Ground return |

> **Wiring tip:** The pull-down resistor is critical — a floating pin generates phantom rising edges that destroy double-click timing. Keep the button wires short to minimise crosstalk. The LED gives a visual flash on each detected double-click.

## Code
```cpp
// Double-Click Event Filter
const int BTN_PIN = 4;
const int LED_PIN = 5;

const unsigned long DEBOUNCE_MS    = 50;    // Mechanical bounce filter
const unsigned long DBLCLICK_MS    = 400;   // Max time between two clicks
const unsigned long SINGLE_WAIT_MS = 450;   // Wait before declaring single-click

bool lastState     = false;
unsigned long lastDebounce  = 0;
unsigned long firstClickTime = 0;
int clickCount = 0;

void setup() {
  pinMode(BTN_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  Serial.begin(115200);
  Serial.println("Double-Click Filter ready.");
}

void loop() {
  bool btn = digitalRead(BTN_PIN);
  unsigned long now = millis();

  // Detect a debounced rising edge
  if (btn && !lastState && (now - lastDebounce > DEBOUNCE_MS)) {
    lastDebounce = now;
    clickCount++;

    if (clickCount == 1) {
      firstClickTime = now;   // Record time of first click
    } else if (clickCount == 2) {
      // Second click — check timing
      if (now - firstClickTime <= DBLCLICK_MS) {
        Serial.println(">> DOUBLE CLICK detected!");
        digitalWrite(LED_PIN, HIGH);
        delay(200);
        digitalWrite(LED_PIN, LOW);
      }
      clickCount = 0;   // Reset after evaluating pair
    }
  }

  // If only one click and the window has expired → single click
  if (clickCount == 1 && (millis() - firstClickTime > SINGLE_WAIT_MS)) {
    Serial.println("Single click.");
    clickCount = 0;
  }

  lastState = btn;
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Button**, and **LED** onto the canvas.
2. Connect Button **output** to **GPIO4**, LED **anode** to **GPIO5**.
3. Paste the code and click **Run**.
4. Click the button widget once — wait → "Single click." appears.
5. Click the button widget twice quickly — "DOUBLE CLICK detected!" appears and the LED flashes.

## Expected Output
Serial Monitor:
```
Double-Click Filter ready.
Single click.
Single click.
>> DOUBLE CLICK detected!
Single click.
>> DOUBLE CLICK detected!
```

## Expected Canvas Behavior
* Single presses produce no LED flash — only a Serial log after the 450 ms window expires.
* Double-clicks within 400 ms flash the LED for 200 ms and print the detection message.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `DEBOUNCE_MS = 50` | Ignores any second rising edge within 50 ms of the first (filters bounce). |
| `DBLCLICK_MS = 400` | Maximum gap between first and second click to be counted as a double-click. |
| `SINGLE_WAIT_MS = 450` | Time the firmware waits after a single press before giving up on a second click and classifying it as single. |
| `clickCount++` | Increments on each valid rising edge — resets to 0 after each resolved event. |
| `if (now - firstClickTime <= DBLCLICK_MS)` | Checks if the second press fell within the double-click window. |

## Hardware & Safety Concept: Time-Window Gesture Recognition
User interface designers often need to distinguish fast sequences of input events from slow ones. The technique is called **time-window event filtering**: a state machine opens a timer window on the first event and waits to see if a second event arrives before the window closes. If it does, the gesture is recognised as a double-event. This same algorithm underlies double-click detection in every computer mouse driver, double-tap detection in smartphone touchscreens, and two-press safety confirmation on industrial machines (press START twice within 1 second to confirm).

## Try This! (Challenges)
1. **Triple-click**: Extend the state machine to detect three clicks within 600 ms.
2. **Different outputs**: Single-click toggles one LED; double-click toggles a second LED.
3. **Adjustable window**: Read a potentiometer on GPIO 34 and map its ADC value to set `DBLCLICK_MS` between 200 and 800 ms dynamically.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Every click registers as double | Bounce events counted as second click | Increase `DEBOUNCE_MS` to 80 ms |
| Double-clicks never detected | Window too short for quick finger | Increase `DBLCLICK_MS` to 600 ms |
| Single-click label delayed | `SINGLE_WAIT_MS` set too high | Reduce to 350 ms for faster response |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [26 - ESP32 Button Press Counter](26-esp32-button-press-counter.md)
- [22 - ESP32 Touch Toggle Controller](22-esp32-touch-toggle-controller.md)
- [16 - ESP32 Button-Controlled LED](16-esp32-button-controlled-led.md)
