# 63 - ESP32 TM1637 Counter Display

Use a push button to increment a counter displayed on a TM1637 4-digit 7-segment display — a physical tally counter with hold-to-reset and overflow wraparound.

## Goal
Learn how to combine button edge detection with a TM1637 display to build a tactile counting device, adding a long-press reset and display wraparound.

## What You Will Build
A push button on GPIO 4 increments a counter shown on the TM1637 display. Holding the button for 2 seconds resets the counter to 0000. The counter wraps from 9999 back to 0000.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| TM1637 4-Digit 7-Segment Display | `tm1637` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| TM1637 Module | VCC | 3V3 | Red | Module power |
| TM1637 Module | GND | GND | Black | Common ground |
| TM1637 Module | CLK | GPIO18 | Yellow | Clock line |
| TM1637 Module | DIO | GPIO19 | Blue | Data line |
| Push Button | Pin 1 | 3V3 | Red | Supply side |
| Push Button | Pin 2 | GPIO4 | Yellow | Signal side |
| 10 kΩ Resistor | Leg 1 | GPIO4 | White | Pull-down resistor |
| 10 kΩ Resistor | Leg 2 | GND | Black | Ground reference |

> **Wiring tip:** The 10 kΩ pull-down holds GPIO 4 LOW when the button is open. On a button press, GPIO 4 goes HIGH. The code detects rising edges for count increments and monitors the duration of button hold for the reset function.

## Code
```cpp
// TM1637 Counter Display — button increment + long-press reset
#include <TM1637Display.h>

const int CLK_PIN = 18;
const int DIO_PIN = 19;
const int BTN_PIN = 4;

TM1637Display display(CLK_PIN, DIO_PIN);

int  counter       = 0;
bool lastBtn       = false;
unsigned long pressStart = 0;

void showCount(int n) {
  display.showNumberDecEx(n, 0b00000000, true);   // No colon, leading zeros
  Serial.print("Count: "); Serial.println(n);
}

void setup() {
  pinMode(BTN_PIN, INPUT);
  display.setBrightness(5);
  display.clear();
  showCount(0);
  Serial.begin(115200);
  Serial.println("Counter Display ready — press button to count.");
}

void loop() {
  bool btn = (digitalRead(BTN_PIN) == HIGH);

  if (btn && !lastBtn) {
    // Rising edge — start timing the press
    pressStart = millis();
  }

  if (!btn && lastBtn) {
    // Falling edge — determine if short or long press
    unsigned long held = millis() - pressStart;

    if (held >= 2000) {
      // Long press (≥ 2 s) — reset counter
      counter = 0;
      Serial.println("RESET!");
    } else {
      // Short press — increment with wraparound
      counter = (counter + 1) % 10000;
    }
    showCount(counter);
  }

  lastBtn = btn;
  delay(20);   // Debounce
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **TM1637 Display**, and **Button** onto the canvas.
2. Connect TM1637 **CLK** → **GPIO18**, **DIO** → **GPIO19**, Button → **GPIO4**.
3. Paste the code and click **Run**.
4. Click the button widget quickly to count; hold 2+ seconds then release to reset.

## Expected Output
Serial Monitor:
```
Counter Display ready — press button to count.
Count: 1
Count: 2
Count: 3
RESET!
Count: 0
Count: 1
```

## Expected Canvas Behavior
* Display shows 0000 at startup.
* Each quick button click increments the displayed count by 1.
* Holding the button widget for 2 seconds then releasing resets to 0000.
* Count wraps from 9999 back to 0000 on the next press.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `pressStart = millis()` | Records the timestamp when the button goes HIGH (rising edge). |
| `unsigned long held = millis() - pressStart` | Calculates how long the button was held when released (falling edge). |
| `if (held >= 2000)` | Classifies a press ≥ 2 seconds as a long press (reset). |
| `counter = (counter + 1) % 10000` | Increments and wraps at 10000 back to 0. |
| `0b00000000` | Second argument to `showNumberDecEx` — no colon segments enabled. |

## Hardware & Safety Concept: Short Press vs Long Press Detection
Detecting both short and long press on a single button is a common embedded UX pattern that doubles the number of functions from one physical button without adding hardware. The technique uses **timing** on the falling edge: measure the total duration from rising edge to falling edge. Thresholds are typically: < 50 ms = bounce (ignore), 50–1000 ms = short press, > 1000 ms = long press. The release event triggers the action (not the press) to allow the firmware to measure hold duration. This pattern is used in: smoke detectors (short press = test, hold 5 s = silence alarm), smart plugs (short = toggle, hold = factory reset), and camera shutters (short = photo, hold = video).

## Try This! (Challenges)
1. **Step mode**: Add a second button on GPIO 5 that decrements the counter (down-counter mode).
2. **Flash on reset**: When long-press reset fires, flash the display three times before showing 0000.
3. **Auto-save display**: Blink the colon every second (toggle `0b01000000` / `0b00000000`) as a "heartbeat" indicator showing the system is alive.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Counter increments multiple times per press | Button bounce | Increase `delay(20)` to `delay(50)` |
| Long-press reset does not fire | `pressStart` not recording correctly | Verify rising edge logic: `if (btn && !lastBtn)` must set `pressStart` |
| Display not updating | Wrong CLK/DIO pin assignment | Double-check CLK → GPIO18 and DIO → GPIO19 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [62 - ESP32 TM1637 7-Segment Display Print Numbers](62-esp32-tm1637-7segment-display-print-numbers.md)
- [26 - ESP32 Button Press Counter](../beginner/26-esp32-button-press-counter.md)
- [27 - ESP32 Double-Click Event Filter](../beginner/27-esp32-double-click-event-filter.md)
