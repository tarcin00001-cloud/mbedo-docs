# 18 - Double Button AND Gate (logical inputs)

Create a physical logical AND gate using two push buttons and an LED on the VEGA ARIES v3 board.

## Goal
Learn how to read multiple digital inputs simultaneously and implement logical AND conjunction logic (`&&`) in code to control an output.

## What You Will Build
Two push buttons are wired to the VEGA ARIES v3 board on pins `GPIO 16` and `GPIO 17` respectively, configured with internal pull-up resistors. An external LED is connected to `GPIO 15`. The LED will only turn ON if both buttons are pressed simultaneously. If only one button is pressed, or neither is pressed, the LED remains OFF.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Push Button (x2) | `button` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 Ω Resistor | `resistor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button 1 | Terminal 1 | GPIO 16 | Blue | Input channel 1 |
| Push Button 1 | Terminal 2 | GND | Black | Ground connection |
| Push Button 2 | Terminal 1 | GPIO 17 | Green | Input channel 2 |
| Push Button 2 | Terminal 2 | GND | Black | Ground connection |
| LED | Anode (+) | GPIO 15 | Red | Digital control signal (through resistor) |
| LED | Cathode (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Both buttons are configured using ARIES internal pull-up resistors, so each button only needs one connection to its respective GPIO pin and one connection to the common ground (GND).

## Code
```cpp
// Double Button AND Gate - VEGA ARIES v3
const int BUTTON1_PIN = 16;
const int BUTTON2_PIN = 17;
const int LED_PIN = 15;

void setup() {
  // Configure both button pins with internal pull-up resistors
  pinMode(BUTTON1_PIN, INPUT_PULLUP);
  pinMode(BUTTON2_PIN, INPUT_PULLUP);
  
  // Configure LED pin as output
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int state1 = digitalRead(BUTTON1_PIN);
  int state2 = digitalRead(BUTTON2_PIN);
  
  // Under INPUT_PULLUP configuration, LOW indicates a pressed button.
  // Both must be LOW for the AND condition to be satisfied.
  if (state1 == LOW && state2 == LOW) {
    digitalWrite(LED_PIN, HIGH);
  } else {
    digitalWrite(LED_PIN, LOW);
  }
  
  delay(20); // Small delay to prevent high frequency polling
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, two **Push Buttons**, an **LED**, and a **220 Ω Resistor** onto the canvas.
2. Wire Button 1 to **GPIO 16** and **GND**.
3. Wire Button 2 to **GPIO 17** and **GND**.
4. Wire the Resistor from **GPIO 15** to the LED's anode (+), and connect the cathode (-) to **GND**.
5. Paste the code into the editor.
6. Click **Run**.
7. Try pressing each button individually on the canvas; the LED should remain OFF.
8. Click and hold both buttons simultaneously (or press one and click the other) to watch the LED turn ON.

## Expected Output
Serial Monitor:
```
System Initialized.
Inputs: BTN1=HIGH, BTN2=HIGH | Output: LOW
Inputs: BTN1=LOW, BTN2=HIGH  | Output: LOW
Inputs: BTN1=LOW, BTN2=LOW   | Output: HIGH
```

## Expected Canvas Behavior
* The external LED remains dark when either button is clicked alone.
* The external LED glows bright red only when both buttons are activated at the same time.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(BUTTON1_PIN, INPUT_PULLUP)` | Activates internal pull-up resistor for Button 1. |
| `pinMode(BUTTON2_PIN, INPUT_PULLUP)` | Activates internal pull-up resistor for Button 2. |
| `state1 == LOW && state2 == LOW` | Performs a logical AND check. Since the pull-ups invert the logic, we check if both signals are LOW. |

## Hardware & Safety Concept
* **Logical Combinational Logic**: Logical gates are the foundation of digital computing. While hardware AND gates require dedicated integrated circuits (like the 74HC08), microcontrollers allow us to implement this logic dynamically in software.
* **Input Isolation**: Because each input pin operates independently, pressing one switch does not affect the voltage or current of the other pin. This provides isolation and prevents feedback loops between different sensor circuits.

## Try This! (Challenges)
1. **NAND Gate Logic**: Modify the code to implement a NAND gate, where the LED is ON by default and turns OFF only when both buttons are pressed.
2. **Serial Print on State Change**: Print a message to the Serial Monitor only when the output state changes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED turns ON when only one button is pressed | Logical error in the code | Verify that the condition checks `state1 == LOW && state2 == LOW` using the `&&` operator |
| LED never turns ON | Incorrect wiring | Make sure the ground connection of both buttons is tied to ARIES GND |
| LED is always ON | Button pins defined as OUTPUT | Ensure pins 16 and 17 are configured as `INPUT_PULLUP` in `setup()` |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [17 - Pull-down button logic (External button with pull-down)](17-pull-down-button-logic.md)
- [19 - Double Button OR Gate](19-double-button-or-gate.md) (Next project)
