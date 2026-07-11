# 50 - Analog Joysticks Button Press (GPIO Digital Input)

Toggle the state of an external LED using the integrated push-button switch on an analog joystick.

## Goal
Understand how push-button switches operate, configure digital inputs with internal pull-up resistors, detect falling-edge transitions, and implement software debounce routines.

## What You Will Build
The switch output (`SW`) of an analog joystick module (or a standalone push-button) is connected to digital input pin `GPIO 16`, and an LED is connected to `GPIO 15`. Pressing the button pulls the input pin to GND, triggering the board to toggle the LED's state.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Joystick Module (with Switch) | `joystick` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 Ω Resistor | `resistor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Joystick Switch | SW | GPIO 16 | Yellow | Input switch signal |
| Joystick Switch | GND | GND | Black | Ground connection |
| LED | Anode (+) | GPIO 15 | Red | Output indicator control (via resistor) |
| LED | Cathode (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** By using `INPUT_PULLUP` mode, the pin is held HIGH inside the SoC. Pressing the button pulls it directly to GND (LOW). Therefore, no external pull-up or pull-down resistor is required for the switch pin.

## Code
```cpp
// Joystick Button Press Toggle - VEGA ARIES v3
const int BUTTON_PIN = 16; // GPIO 16
const int LED_PIN = 15;    // GPIO 15

// Variables to keep track of button transitions
int lastButtonState = HIGH; // Starts HIGH due to INPUT_PULLUP
int ledState = LOW;

void setup() {
  // Configure button pin with the internal pull-up resistor enabled
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  
  // Configure LED pin as digital output
  pinMode(LED_PIN, OUTPUT);
  
  // Initialize serial communication
  Serial.begin(115200);
}

void loop() {
  // Read current button state (LOW = pressed, HIGH = idle)
  int buttonState = digitalRead(BUTTON_PIN);
  
  // Detect transition from HIGH to LOW (falling edge, button pressed)
  if (buttonState == LOW && lastButtonState == HIGH) {
    // Toggle the LED state
    if (ledState == LOW) {
      ledState = HIGH;
    } else {
      ledState = LOW;
    }
    
    // Write state to the LED
    digitalWrite(LED_PIN, ledState);
    
    // Print status
    Serial.print("Button Pressed! LED Toggled to: ");
    Serial.println(ledState);
    
    // Debounce delay to prevent multiple triggers from electrical noise
    delay(150);
  }
  
  // Save current button state for next loop iteration
  lastButtonState = buttonState;
  
  // Polling rate delay
  delay(20);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, an **LED**, a **220 Ω Resistor**, and a **Joystick** onto the canvas.
2. Wire the Joystick's SW pin to **GPIO 16** and the GND pin to **GND**.
3. Wire the LED's anode through the resistor to **GPIO 15** and the cathode to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Click the joystick button (or SW pin widget) on the canvas to toggle the LED.

## Expected Output
Serial Monitor:
```
System Initialized.
Button Pressed! LED Toggled to: 1
Button Pressed! LED Toggled to: 0
```

## Expected Canvas Behavior
* The warning LED toggles ON or OFF on the canvas each time the joystick switch button is clicked.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(BUTTON_PIN, INPUT_PULLUP)` | Configures the input pin with the internal pull-up resistor active, keeping the default state HIGH. |
| `if (buttonState == LOW && lastButtonState == HIGH)` | Captures the transition from HIGH to LOW (when the button is pressed). |
| `delay(150)` | Software debounce delay to ignore electrical noise during contact changes. |

## Hardware & Safety Concept: Internal Pull-up Resistors and Contact Bounce
* **Internal Pull-up Resistors**: Microcontrollers have internal resistors (typically 20 kΩ to 50 kΩ) that can be enabled in software to pull input pins to VCC. This prevents the input pin from "floating" (randomly picking up electrical noise) when the button is open, eliminating external resistors from the circuit.
* **Contact Bounce**: Tactile switches are mechanical devices. When pressed, the internal metal contacts bounce against each other, making and breaking contact multiple times within a few milliseconds. A microcontroller is fast enough to read these bounces as multiple separate presses. Software debouncing works by ignoring any subsequent input transitions for a short window (e.g. 150 ms) after the first state change is registered.

## Try This! (Challenges)
1. **Buzzer Click Feedback**: Connect the active buzzer to `GPIO 14`. Sound a short, sharp 20 ms click beep every time the button is successfully pressed.
2. **Double Press Detection**: Keep track of the time elapsed since the last button press using `millis()`. If two presses occur within 300 ms, flash the onboard RGB LED Red, Blue, and Green in quick succession.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED toggles erratically or multiple times on a single press | Debounce time too short | Increase the debounce delay from 150 ms to 250 ms in code |
| LED stays ON or does not toggle | Button is wired incorrectly or floating input pin | Check that the code uses `INPUT_PULLUP` instead of `INPUT`. If using a standalone button, verify it is connected between GPIO 16 and GND |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [49 - Sound sensor threshold alarm](49-sound-sensor-threshold-alarm.md) (Previous project)
- [01 - Onboard RGB LED Blink (Red channel)](01-onboard-rgb-led-blink-red.md) (Cycle start)
