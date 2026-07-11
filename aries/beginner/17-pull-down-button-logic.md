# 17 - Pull-down button logic (External button with pull-down)

Control an external LED based on the status of a push button configured with an external pull-down resistor.

## Goal
Understand the electrical concept of external pull-down resistors, configure a GPIO input pin without internal pull-ups, and read digital states where high voltage represents a button press.

## What You Will Build
A push button is wired to `GPIO 16` with an external 10 kΩ pull-down resistor. When the button is not pressed, the resistor ties the input pin to GND (LOW). When pressed, the button connects the input pin directly to 3.3V (HIGH). An external LED on `GPIO 15` lights up when the button is held down and turns off when released.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 Ω Resistor | `resistor` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button | Terminal 1 | 3.3V | Red | Power source for button press |
| Push Button | Terminal 2 | GPIO 16 | Blue | Input sense node connected to ARIES |
| 10 kΩ Resistor | Lead 1 | GPIO 16 | Blue | Connected to the button output terminal |
| 10 kΩ Resistor | Lead 2 | GND | Black | Pulls input to ground when button is open |
| LED | Anode (+) | GPIO 15 | Red | Digital control signal (through resistor) |
| LED | Cathode (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect one terminal of the switch to 3.3V. Connect the other terminal of the switch to both the ARIES digital input pin (GPIO 16) and one end of the 10 kΩ resistor. Connect the other end of the 10 kΩ resistor to GND.

## Code
```cpp
// External Pull-Down Button Logic - VEGA ARIES v3
const int BUTTON_PIN = 16;
const int LED_PIN = 15;

void setup() {
  // Configure the button pin as standard input (no internal pull-up)
  pinMode(BUTTON_PIN, INPUT);
  
  // Configure the LED pin as output
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int buttonState = digitalRead(BUTTON_PIN);
  
  // In a pull-down configuration, HIGH means the button is pressed
  if (buttonState == HIGH) {
    digitalWrite(LED_PIN, HIGH);
  } else {
    digitalWrite(LED_PIN, LOW);
  }
  
  delay(10); // Small delay to stabilize readings
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, a **Push Button**, an **LED**, a **220 Ω Resistor**, and a **10 kΩ Resistor** onto the canvas.
2. Connect Terminal 1 of the button to **3.3V**.
3. Connect Terminal 2 of the button to **GPIO 16**.
4. Connect one end of the **10 kΩ Resistor** to the connection between Terminal 2 and **GPIO 16**, and the other end to **GND**.
5. Connect one end of the **220 Ω Resistor** to **GPIO 15** and the other to the LED's anode (+).
6. Connect the LED's cathode (-) to **GND**.
7. Paste the code into the editor.
8. Click **Run**.
9. Click and hold the Push Button on the canvas; verify that the LED remains lit while holding, and shuts off when released.

## Expected Output
Serial Monitor:
```
System Initialized.
GPIO 16 State: LOW
GPIO 16 State: HIGH
GPIO 15 State: HIGH
GPIO 16 State: LOW
GPIO 15 State: LOW
```

## Expected Canvas Behavior
* The external LED lights up while the button is pressed/clicked, and immediately dims when the button is released.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(BUTTON_PIN, INPUT)` | Configures GPIO 16 as a high-impedance input pin without activating any internal resistors. |
| `digitalRead(BUTTON_PIN)` | Samples the voltage on GPIO 16 (resolving to HIGH or LOW). |
| `buttonState == HIGH` | Checks if the input voltage is close to 3.3V, indicating a closed circuit through the switch. |

## Hardware & Safety Concept
* **Pull-down Resistors**: Without a pull-down resistor, when the mechanical switch is open, the input pin is electrically disconnected (floating). In this state, electrostatic noise in the air can cause the pin to randomly toggle between HIGH and LOW. The 10 kΩ pull-down resistor ensures a weak path to ground, draining any stray charge and keeping the pin at 0V (LOW).
* **Current Draw through Switch**: When the button is pressed, a small current flows from the 3.3V rail through the 10 kΩ resistor to GND. According to Ohm's Law ($I = \frac{V}{R}$), this current is $3.3\text{V} / 10\text{k}\Omega = 0.33\text{ mA}$, which is extremely safe and prevents shorting the power supply.

## Try This! (Challenges)
1. **Latching Toggle Logic**: Modify the code using variables (similar to Project 16) to make this external pull-down button toggle the LED state permanently on each press, rather than only when held.
2. **Inverted LED Output**: Modify the code so the LED is normally ON and turns OFF only when the button is pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays on constantly | Pull-down resistor disconnected | Verify that the 10 kΩ resistor connects GPIO 16 directly to GND |
| LED is very dim | LED resistor too large | Verify that you are using a 220 Ω resistor for the LED, and 10 kΩ only for the button pull-down |
| No response from button | Button wired incorrectly | Ensure one side of the button is connected to 3.3V, not GND |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [16 - Push Button LED toggle (internal pull-up)](16-push-button-led-toggle.md)
- [18 - Double Button AND Gate (logical inputs)](18-double-button-and-gate.md) (Next project)
