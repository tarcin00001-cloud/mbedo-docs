# 16 - Push Button LED toggle (internal pull-up)

Toggle the state of an external LED on the VEGA ARIES v3 board using a push button configured with the internal pull-up resistor.

## Goal
Learn how to configure a GPIO pin as a digital input with an internal pull-up resistor, detect button state transitions (falling edge), and toggle a digital output state.

## What You Will Build
A push button is connected to pin `GPIO 16` and configured with an internal pull-up resistor. When the button is pressed, the signal changes from HIGH to LOW (falling edge). The VEGA ARIES v3 board detects this press and toggles the state of an external LED connected to `GPIO 15` (ON if it was OFF, and vice-versa).

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

> **Wiring tip:** The push button does not need an external resistor because we are configuring the ARIES pin with an internal pull-up resistor. Make sure the LED has a 220 Ω resistor in series to limit the current.

## Code
```cpp
// Push Button LED Toggle (Internal Pull-Up) - VEGA ARIES v3
const int BUTTON_PIN = 16;
const int LED_PIN = 15;

int lastButtonState = HIGH; // Starts HIGH due to the internal pull-up
int ledState = LOW;         // Keep track of the LED state

void setup() {
  // Configure the button pin as input with internal pull-up
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  
  // Configure the LED pin as output
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, ledState);
}

void loop() {
  int currentButtonState = digitalRead(BUTTON_PIN);
  
  // Check for transition from HIGH to LOW (button press)
  if (lastButtonState == HIGH && currentButtonState == LOW) {
    ledState = !ledState;
    digitalWrite(LED_PIN, ledState);
    delay(50); // Debounce delay
  }
  
  lastButtonState = currentButtonState;
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, a **Push Button**, an **LED**, and a **220 Ω Resistor** onto the canvas.
2. Wire Terminal 1 of the Push Button to **GPIO 16**.
3. Wire Terminal 2 of the Push Button to **GND**.
4. Wire one end of the Resistor to **GPIO 15** and the other end to the LED's anode (+).
5. Wire the LED's cathode (-) to **GND**.
6. Paste the code into the editor.
7. Click **Run**.
8. Click the Push Button on the canvas and observe the LED toggling ON and OFF.

## Expected Output
Serial Monitor:
```
System Initialized.
GPIO 16 State: HIGH
GPIO 16 State: LOW
GPIO 15 State: HIGH
GPIO 16 State: HIGH
GPIO 16 State: LOW
GPIO 15 State: LOW
```

## Expected Canvas Behavior
* Pressing the push button causes the external LED to turn ON if it was OFF, or turn OFF if it was ON.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(BUTTON_PIN, INPUT_PULLUP)` | Configures GPIO 16 as an input pin with the internal pull-up resistor enabled, keeping the input state HIGH when the button is not pressed. |
| `digitalRead(BUTTON_PIN)` | Reads the digital value of the button pin. |
| `lastButtonState == HIGH && currentButtonState == LOW` | Detects a falling edge, indicating that the button was pressed. |
| `ledState = !ledState` | Inverts the current LED state variable. |
| `delay(50)` | Introduces a 50 ms debounce delay to ignore electrical noise during contact closing. |

## Hardware & Safety Concept
* **Internal Pull-up Resistors**: Microcontroller input pins can float randomly between HIGH and LOW states if left unconnected. Pull-up resistors resolve this by pulling the voltage up to the 3.3V rail. The THEJAS32 SoC has built-in configurable pull-up resistors (around 30 kΩ to 50 kΩ) that save space and wiring on breadboards.
* **Debouncing**: Mechanical switches contain internal spring contacts that bounce when pressed, causing multiple false signal transitions (chatter). Software debouncing using a small delay ignores these rapid transitions and ensures that only one toggle action is registered per physical press.

## Try This! (Challenges)
1. **Change Debounce Time**: Modify the delay value and observe if the button toggle becomes unreliable or unresponsive.
2. **Reverse State Logic**: Change the code to toggle the LED when the button is released (rising edge, i.e., transitioning from LOW to HIGH).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED toggles randomly or flickers | Missing debounce | Ensure `delay(50)` is present in the falling-edge block |
| LED never turns ON | Pin definition error or incorrect wiring | Check that button is on GPIO 16 and LED anode is connected to GPIO 15 via a resistor |
| LED stays ON | Incorrect edge detection | Check that you are comparing `lastButtonState` and `currentButtonState` correctly |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [15 - RGB LED breathing cycle](15-rgb-led-breathing-cycle.md)
- [17 - Pull-down button logic (External button with pull-down)](17-pull-down-button-logic.md) (Next project)
