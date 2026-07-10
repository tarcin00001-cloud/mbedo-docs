# 17 - ESP32 Pull-up Button Logic

Use the ESP32's built-in internal pull-up resistor to read a push button with inverted logic — no external resistors required.

## Goal
Understand how internal pull-up resistors work and why button state reads as `LOW` when pressed and `HIGH` when released when using `INPUT_PULLUP` mode.

## What You Will Build
A push button wired between GPIO 4 and GND. When pressed, the internal pull-up is pulled to GND (LOW). The code lights an LED on GPIO 5 whenever the button is pressed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| LED (any colour) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button | Pin 1 (one side) | GPIO4 | Yellow | Input signal — internal pull-up active |
| Push Button | Pin 2 (other side) | GND | Black | Pulls pin LOW when button pressed |
| LED | Anode (+) | GPIO5 via 330 Ω | Orange | Current-limited output |
| LED | Cathode (−) | GND | Black | Ground return |

> **Wiring tip:** With `INPUT_PULLUP`, no external resistor is needed on the button. The ESP32 contains a ~45 kΩ resistor internally between the pin and 3V3. Connecting the other button leg to GND pulls the signal to LOW when pressed.

## Code
```cpp
// Pull-up Button Logic — button to GND, LED on GPIO 5
const int BTN_PIN = 4;
const int LED_PIN = 5;

void setup() {
  // INPUT_PULLUP enables the internal ~45 kΩ resistor to 3V3
  pinMode(BTN_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);

  Serial.begin(115200);
  Serial.println("Pull-up Button Logic ready.");
}

void loop() {
  // With INPUT_PULLUP: LOW means pressed, HIGH means released
  int btnState = digitalRead(BTN_PIN);

  if (btnState == LOW) {
    // Button is pressed (pin pulled to GND)
    digitalWrite(LED_PIN, HIGH);
    Serial.println("Button PRESSED — LED ON");
  } else {
    // Button is released (pin held HIGH by internal pull-up)
    digitalWrite(LED_PIN, LOW);
    Serial.println("Button RELEASED — LED OFF");
  }

  delay(100);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Button**, and **LED** onto the canvas.
2. Connect Button **output** to **GPIO4** (MbedO applies internal pull-up automatically when `INPUT_PULLUP` is used).
3. Connect LED **anode** to **GPIO5** and **cathode** to **GND**.
4. Paste the C++ code and click **Run**.
5. Press the button widget — the LED lights up.

## Expected Output
Serial Monitor:
```
Pull-up Button Logic ready.
Button RELEASED — LED OFF
Button PRESSED — LED ON
Button PRESSED — LED ON
Button RELEASED — LED OFF
```

## Expected Canvas Behavior
* When the button widget is NOT pressed, the LED is off.
* Clicking and holding the button lights the LED.
* Serial Monitor prints the current state every 100 ms.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `pinMode(BTN_PIN, INPUT_PULLUP)` | Enables the internal pull-up resistor — pin reads HIGH at rest. |
| `if (btnState == LOW)` | Detects a press because pressing pulls the pin to GND (LOW). |
| `digitalWrite(LED_PIN, HIGH)` | Turns the LED on during the active-LOW press event. |

## Hardware & Safety Concept: Internal Pull-up Resistors
Modern microcontrollers include resistors inside the chip that connect digital input pins to the supply voltage (3V3 on the ESP32). When you call `pinMode(pin, INPUT_PULLUP)` you engage this internal resistor. The pin is therefore weakly held HIGH. Connecting a button to GND gives the signal a short path to ground when pressed — strong enough to override the internal pull-up. This is called **active-LOW** logic because the active event (pressing the button) produces a LOW reading. The advantage: one fewer component (no external resistor), saving board space and assembly time.

## Try This! (Challenges)
1. **Inverted display**: Print "ACTIVE" when the reading is LOW and "IDLE" when HIGH.
2. **LED opposite**: Wire the logic so the LED is on all the time and turns OFF when the button is pressed.
3. **Debounced read**: Add a second `digitalRead` 10 ms after the first; only act if both readings agree.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED is always ON | Button wired to 3V3 instead of GND | Move one button leg to GND |
| LED never responds | Wrong pin number | Check `BTN_PIN` matches your actual wiring |
| Sporadic extra readings | Contact bounce | Increase the `delay(100)` or add a debounce check |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [16 - ESP32 Button-Controlled LED](16-esp32-button-controlled-led.md)
- [18 - ESP32 Pull-down Button Logic](18-esp32-pull-down-button-logic.md)
- [27 - ESP32 Double-Click Event Filter](27-esp32-double-click-event-filter.md)
