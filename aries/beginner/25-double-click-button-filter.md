# 25 - Double click button filter

Detect a double click on a single push button within a 500 ms window to toggle an external LED on the VEGA ARIES v3 board.

## Goal
Learn how to use non-blocking timer logic (`millis()`) to track time differences, build a time-dependent state machine, and filter out single clicks to detect double clicks.

## What You Will Build
A push button is connected to `GPIO 16` and configured with the internal pull-up resistor. An external LED is connected to `GPIO 15`. When you press the button once, nothing happens immediately. If you press it a second time within 500 milliseconds, a double click is detected, and the external LED toggles state (ON to OFF, or OFF to ON). If the second click does not happen within 500 ms, the system resets.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 Ω Resistor | `resistor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button | Terminal 1 | GPIO 16 | Blue | Digital input signal pin |
| Push Button | Terminal 2 | GND | Black | Ground connection |
| LED | Anode (+) | GPIO 15 | Red | Digital control signal (through resistor) |
| LED | Cathode (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Wiring remains identical to the single button toggle, but the logic is handled entirely in software by tracking elapsed milliseconds.

## Code
```cpp
// Double Click Button Filter - VEGA ARIES v3
const int BUTTON_PIN = 16;
const int LED_PIN = 15;

int lastButtonState = HIGH;      // Tracks previous button state
int ledState = LOW;              // Track current LED status
int clickCount = 0;              // Count button presses in the active window
unsigned long lastClickTime = 0; // Timestamp of the first click

void setup() {
  // Configure button as input with internal pull-up
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  
  // Configure LED as output
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, ledState);
}

void loop() {
  int currentButtonState = digitalRead(BUTTON_PIN);
  unsigned long currentTime = millis(); // Get the elapsed time since startup
  
  // Detect falling edge (button press)
  if (lastButtonState == HIGH && currentButtonState == LOW) {
    if (clickCount == 0) {
      // First click of a potential double click
      clickCount = 1;
      lastClickTime = currentTime;
    } 
    else if (clickCount == 1) {
      // Second click check
      if (currentTime - lastClickTime <= 500) {
        // Double click detected within 500 ms window!
        ledState = !ledState;
        digitalWrite(LED_PIN, ledState);
        clickCount = 0; // Reset count
      } else {
        // Too slow. Treat this click as a new first click
        lastClickTime = currentTime;
      }
    }
    delay(150); // Software debounce
  }
  
  // Timeout check: If 500 ms passes since the first click without a second click, reset
  if (clickCount == 1 && (currentTime - lastClickTime > 500)) {
    clickCount = 0;
  }
  
  lastButtonState = currentButtonState;
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, a **Push Button**, an **LED**, and a **220 Ω Resistor** onto the canvas.
2. Wire the Push Button between **GPIO 16** and **GND**.
3. Wire the **220 Ω Resistor** in series between **GPIO 15** and the LED's anode (+). Connect the LED's cathode (-) to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Click the Push Button once. Observe that the LED stays OFF.
7. Click the button twice quickly (within half a second). Observe that the LED toggles ON.
8. Click the button twice quickly again. Observe that the LED toggles OFF.

## Expected Output
Serial Monitor:
```
System Initialized.
First Click Detected. Waiting for second click...
Double Click Confirmed! Toggling LED state.
GPIO 15 State: HIGH
First Click Detected. Waiting for second click...
Timeout. Resetting click count.
```

## Expected Canvas Behavior
* Single button clicks do not toggle the LED.
* Rapid double clicks toggle the external LED state reliably.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `unsigned long currentTime = millis()` | Reads the number of milliseconds elapsed since the ARIES board booted up. |
| `clickCount == 0` | Checks if we are currently listening for the very first click in a sequence. |
| `currentTime - lastClickTime <= 500` | Calculates the duration between the first and second clicks to verify it fits the 500 ms double click criteria. |
| `clickCount = 0` | Resets the state machine counters once a double click is completed or when a timeout occurs. |

## Hardware & Safety Concept
* **Non-blocking Timing (`millis()`)**: Standard microcontrollers run sequentially. Using `delay(500)` blocks the processor from reading any inputs during that period. By using `millis()`, we can check the time difference non-blockingly, allowing the processor to continue checking inputs in real-time.
* **Overcoming Arithmetic Overflow**: The `millis()` timer runs on an unsigned 32-bit integer that overflows and resets to 0 after approximately 50 days. By subtracting the timestamps using unsigned arithmetic (`currentTime - lastClickTime`), the calculation remains correct even across overflows.

## Try This! (Challenges)
1. **Triple Click Detector**: Extend the state machine logic so that the LED only toggles if you click the button three times within a 1-second window.
2. **Speed Dial Adjustable Window**: Connect a potentiometer on pin `ADC0` and map its value (0–1023) to represent the click window duration (e.g. 200 ms to 1000 ms), allowing you to adjust double click speed on the fly.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Double clicks are not registered | Window too narrow or debounce too high | Decrease the debounce `delay(150)` to `delay(100)` or increase the timeout window from `500` to `700` |
| Single clicks toggle the LED | Debounce chatter | Ensure the debounce delay is not missing. Mechanical bounce might trigger two fast clicks on one press |
| No toggle occurs | Button wired to wrong pin | Confirm the switch is on GPIO 16 and the LED is on GPIO 15 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [24 - Button counter (output logs to Serial)](24-button-counter.md)
- [26 - Tilt sensor alarm](26-tilt-sensor-alarm-sequence.md) (Next project)
