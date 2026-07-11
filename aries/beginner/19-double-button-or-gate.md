# 19 - Double Button OR Gate

Create a physical logical OR gate using two push buttons and an LED on the VEGA ARIES v3 board.

## Goal
Learn how to read multiple digital inputs and implement logical OR disjunction logic (`||`) in code to control an output.

## What You Will Build
Two push buttons are wired to the VEGA ARIES v3 board on pins `GPIO 16` and `GPIO 17` respectively, configured with internal pull-up resistors. An external LED is connected to `GPIO 15`. The LED will turn ON if either button is pressed, or if both buttons are pressed simultaneously. The LED turns OFF only when both buttons are released.

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
// Double Button OR Gate - VEGA ARIES v3
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
  // Either being LOW satisfies the OR condition.
  if (state1 == LOW || state2 == LOW) {
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
7. Try pressing each button individually on the canvas; the LED should turn ON for either press.
8. Click and hold both buttons simultaneously to watch the LED remain ON.
9. Release both buttons to see the LED turn OFF.

## Expected Output
Serial Monitor:
```
System Initialized.
Inputs: BTN1=HIGH, BTN2=HIGH | Output: LOW
Inputs: BTN1=LOW, BTN2=HIGH  | Output: HIGH
Inputs: BTN1=HIGH, BTN2=LOW  | Output: HIGH
Inputs: BTN1=LOW, BTN2=LOW   | Output: HIGH
```

## Expected Canvas Behavior
* The external LED turns ON when either button 1, button 2, or both are clicked/activated.
* The LED goes dark only when neither button is clicked.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(BUTTON1_PIN, INPUT_PULLUP)` | Activates internal pull-up resistor for Button 1. |
| `pinMode(BUTTON2_PIN, INPUT_PULLUP)` | Activates internal pull-up resistor for Button 2. |
| `state1 == LOW || state2 == LOW` | Performs a logical OR check. If either input is LOW, the expression evaluates to true. |

## Hardware & Safety Concept
* **Logical Combinational Logic (OR)**: The OR gate represents a parallel decision-making path. In security systems, this is equivalent to having multiple sensor triggers where any single trigger can sound the alarm (output).
* **Power Bus Stability**: When driving multiple inputs, current consumption remains low as long as the inputs are high-impedance. The external LED draws around 10 mA when lit, which is within the safe drive limits of the THEJAS32 SoC.

## Try This! (Challenges)
1. **NOR Gate Logic**: Modify the code to implement a NOR gate, where the LED is ON by default and turns OFF if either button (or both) is pressed.
2. **Alert Active Pin**: Print which button was pressed (e.g. "Button 1 pressed" or "Button 2 pressed") to the Serial Monitor.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays ON continuously | Button pin grounded | Check if GPIO 16 or 17 is shorted directly to GND without going through the switch |
| LED never turns ON | Broken LED or resistor mismatch | Verify that the LED is connected correctly (anode to GPIO 15, cathode to GND) and that a 220 Ω resistor is used |
| LED only turns ON when BOTH buttons are pressed | Logical operator error | Ensure you are using the OR operator `||` instead of the AND operator `&&` |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [18 - Double Button AND Gate (logical inputs)](18-double-button-and-gate.md)
- [20 - TTP223 Touch Lamp](20-ttp223-touch-lamp.md) (Next project)
