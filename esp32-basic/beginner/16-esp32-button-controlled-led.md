# 16 - ESP32 Button-Controlled LED

Control an external LED using a push button wired to the ESP32 with no internal pull resistors.

## Goal
Learn how to read a digital input pin from a physical push button and use its state to switch an LED on and off in real time.

## What You Will Build
A push button connected to GPIO 4 controls an external LED on GPIO 5. While the button is held down the LED lights up; releasing the button turns the LED off.

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
| Push Button | Pin 1 (one side) | GPIO4 | Yellow | Input signal pin |
| Push Button | Pin 2 (other side) | 3V3 | Red | Pulls pin HIGH when pressed |
| 10 kΩ Resistor | Leg 1 | GPIO4 | White | Pull-down to ground |
| 10 kΩ Resistor | Leg 2 | GND | Black | Ground reference |
| LED | Anode (+) | GPIO5 via 330 Ω | Orange | Current-limited output |
| LED | Cathode (−) | GND | Black | Ground return |

> **Wiring tip:** The 10 kΩ pull-down resistor keeps GPIO 4 at a known LOW state when the button is not pressed. Without it the pin would float and produce random readings. The 330 Ω resistor on the LED limits current to a safe ~10 mA.

## Code
```cpp
// Button-Controlled LED
const int BTN_PIN = 4;   // Input: push button
const int LED_PIN = 5;   // Output: external LED

void setup() {
  pinMode(BTN_PIN, INPUT);   // External pull-down resistor fitted
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);

  Serial.begin(115200);
  Serial.println("Button-Controlled LED ready.");
}

void loop() {
  // Read the button state (HIGH when pressed, LOW when released)
  int state = digitalRead(BTN_PIN);
  digitalWrite(LED_PIN, state);

  if (state == HIGH) {
    Serial.println("Button PRESSED  — LED ON");
  } else {
    Serial.println("Button RELEASED — LED OFF");
  }

  delay(100); // Debounce pause
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Button**, and **LED** onto the canvas.
2. Connect Button **output** to **GPIO4**.
3. Connect LED **anode** to **GPIO5** and **cathode** to **GND**.
4. Paste the C++ code into the editor.
5. Select interpreted C++ mode and click **Run**.
6. Click the button widget on the canvas to toggle the LED.

## Expected Output
Serial Monitor:
```
Button-Controlled LED ready.
Button RELEASED — LED OFF
Button PRESSED  — LED ON
Button PRESSED  — LED ON
Button RELEASED — LED OFF
```

## Expected Canvas Behavior
* While the button widget is held, the LED component lights up.
* Releasing the button extinguishes the LED immediately.
* The Serial Monitor updates every 100 ms reflecting the current button state.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `pinMode(BTN_PIN, INPUT)` | Configures GPIO 4 as a plain digital input — an external resistor provides the pull-down. |
| `int state = digitalRead(BTN_PIN)` | Reads the logic level: `1` (HIGH) when pressed, `0` (LOW) when released. |
| `digitalWrite(LED_PIN, state)` | Mirrors the button state directly to the LED pin — no `if` needed for the output. |
| `delay(100)` | 100 ms pause suppresses mechanical contact bounce from producing rapid flicker. |

## Hardware & Safety Concept: Pull-Down Resistors
A floating digital input is an input pin that is not connected to any defined voltage level. When a button is simply wired between 3V3 and a pin, releasing the button leaves the pin entirely disconnected — it picks up electrical noise and reads random HIGH/LOW values. A **pull-down resistor** (typically 10 kΩ) connects the input pin to GND through a high resistance. When the button is open the resistor holds the pin firmly at 0 V (LOW). When the button is pressed, the 3V3 supply easily overpowers the 10 kΩ resistor, pulling the pin to HIGH. This gives a clean, predictable signal with no floating.

## Try This! (Challenges)
1. **Inverse Logic**: Rewire so the LED turns OFF when the button is pressed and ON when released. Hint: use `!state`.
2. **Hold Duration Log**: Start a timer when the button is pressed and print how long it was held when released.
3. **Blink on Press**: Instead of a steady light, make the LED blink rapidly while the button is held.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED flickers when button not pressed | Floating input — no pull-down | Verify the 10 kΩ resistor is connected between GPIO 4 and GND |
| LED stays ON regardless of button | Short circuit or wrong pin | Confirm GPIO 4 reads LOW when button is released in Serial Monitor |
| No LED response | LED wired backwards | Swap LED anode and cathode leads |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [17 - ESP32 Pull-up Button Logic](17-esp32-pull-up-button-logic.md)
- [18 - ESP32 Pull-down Button Logic](18-esp32-pull-down-button-logic.md)
- [26 - ESP32 Button Press Counter](26-esp32-button-press-counter.md)
