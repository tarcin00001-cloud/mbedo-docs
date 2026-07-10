# 18 - ESP32 Pull-down Button Logic

Wire a push button with an external pull-down resistor and use `INPUT` mode to achieve active-HIGH button reads.

## Goal
Understand external pull-down wiring, compare it with the internal pull-up approach from Project 17, and confirm that active-HIGH logic produces `HIGH` on press and `LOW` on release.

## What You Will Build
A push button between 3V3 and GPIO 4, with a 10 kΩ resistor from GPIO 4 to GND. Pressing the button drives the pin HIGH. The code lights an LED on GPIO 5 when the button is pressed.

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
| Push Button | Pin 1 (one side) | 3V3 | Red | Supply voltage feeds pin when button pressed |
| Push Button | Pin 2 (other side) | GPIO4 | Yellow | Signal input pin |
| 10 kΩ Resistor | Leg 1 | GPIO4 | White | Pull-down: keeps pin LOW when button open |
| 10 kΩ Resistor | Leg 2 | GND | Black | Ground reference |
| LED | Anode (+) | GPIO5 via 330 Ω | Orange | Current-limited output |
| LED | Cathode (−) | GND | Black | Ground return |

> **Wiring tip:** The 10 kΩ pull-down resistor must connect directly between GPIO 4 and GND on your breadboard. Without it, releasing the button leaves GPIO 4 floating and the reading becomes unpredictable. The 330 Ω resistor on the LED keeps current below 10 mA.

## Code
```cpp
// Pull-down Button Logic — button to 3V3, 10k pull-down on pin
const int BTN_PIN = 4;
const int LED_PIN = 5;

void setup() {
  // Plain INPUT — external pull-down resistor handles the bias
  pinMode(BTN_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);

  Serial.begin(115200);
  Serial.println("Pull-down Button Logic ready.");
}

void loop() {
  // Active-HIGH: HIGH when pressed, LOW when released
  int btnState = digitalRead(BTN_PIN);

  if (btnState == HIGH) {
    // Button pressed — 3V3 drives pin HIGH
    digitalWrite(LED_PIN, HIGH);
    Serial.println("Button PRESSED — LED ON");
  } else {
    // Button released — pull-down holds pin LOW
    digitalWrite(LED_PIN, LOW);
    Serial.println("Button RELEASED — LED OFF");
  }

  delay(100);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Button**, and **LED** onto the canvas.
2. Connect Button **output** to **GPIO4** (MbedO simulates the pull-down bias).
3. Connect LED **anode** to **GPIO5** and **cathode** to **GND**.
4. Paste the C++ code and click **Run**.
5. Press the button widget to turn the LED on.

## Expected Output
Serial Monitor:
```
Pull-down Button Logic ready.
Button RELEASED — LED OFF
Button PRESSED — LED ON
Button PRESSED — LED ON
Button RELEASED — LED OFF
```

## Expected Canvas Behavior
* LED is off when the button is not pressed (pin reads LOW via pull-down).
* LED lights up immediately on button press (pin reads HIGH from 3V3).
* Serial Monitor updates every 100 ms.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `pinMode(BTN_PIN, INPUT)` | Configures the pin as a plain digital input; external resistor provides the pull-down. |
| `if (btnState == HIGH)` | Detects a press because pressing connects the pin to 3V3 (HIGH). |
| `digitalWrite(LED_PIN, HIGH)` | Turns the LED on during the active-HIGH press event. |
| `delay(100)` | Short debounce pause to filter mechanical contact bounce. |

## Hardware & Safety Concept: Active-HIGH vs Active-LOW Logic
The choice between pull-up (active-LOW) and pull-down (active-HIGH) wiring changes the logic polarity of your circuit. Active-LOW circuits read `LOW` on an event and `HIGH` at rest — common in professional electronics because it matches failsafe behaviour (if a wire breaks, the system stays in the safe HIGH/idle state). Active-HIGH circuits read `HIGH` on an event and `LOW` at rest — more intuitive for beginners because the code checks `if (state == HIGH)` to detect a press. Both work identically in practice; you must simply match your code logic to your wiring. The ESP32 supports both modes: use `INPUT_PULLUP` for active-LOW, and `INPUT` with an external pull-down for active-HIGH.

## Try This! (Challenges)
1. **Compare outputs**: Wire Project 17 and Project 18 side by side with two buttons and two LEDs; verify the logic polarity difference.
2. **LED on-by-default**: Start with the LED on and turn it off on button press, requiring you to invert the condition.
3. **Serial state machine**: Print `IDLE → ACTIVE → IDLE` as the button transitions rather than repeating the state.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED flickers when button not pressed | Missing pull-down resistor | Add a 10 kΩ resistor between GPIO 4 and GND |
| LED never turns on | Button wired to GND instead of 3V3 | Move the supply side of the button to 3V3 |
| LED dims instead of full brightness | Wrong LED resistor value | Use a 330 Ω resistor, not a higher value |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [16 - ESP32 Button-Controlled LED](16-esp32-button-controlled-led.md)
- [17 - ESP32 Pull-up Button Logic](17-esp32-pull-up-button-logic.md)
- [19 - ESP32 Double Button AND Logic](19-esp32-double-button-and-logic.md)
