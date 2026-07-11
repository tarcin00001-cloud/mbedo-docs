# 135 - Robotic Arm Position Memory Log

Read joint angles from three potentiometers representing robotic arm joints, and log coordinates to the Serial Monitor upon pressing a button on GPIO 16.

## Goal
Learn how to interface input buttons using pull-up resistors, implement loop-free state change detection (edge detection) to capture events, and log multi-variable coordinate records.

## What You Will Build
A teach-mode coordinate logger for a robotic arm. Three potentiometers act as joint angle sensors connected to `ADC0` (GP26), `ADC1` (GP27), and `ADC2` (GP28). A push button is connected to `GPIO 16`. When the button is pressed, the ARIES board reads the current positions of the three joints, maps them to angles (0° to 180°), logs the formatted coordinate packet to the Serial Monitor, and debounces the button press to avoid multiple logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| 3x Potentiometers | `potentiometer` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| 10k-ohm Resistor (for Button) | `resistor` | Optional | Yes (if external pull-up used) |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer 1 (J1) | VCC / GND | 3V3 / GND | Red / Black | J1 sensor power |
| Potentiometer 1 (J1) | Output / Wiper | ADC0 (GP26) | Yellow | Joint 1 sensor feedback |
| Potentiometer 2 (J2) | VCC / GND | 3V3 / GND | Red / Black | J2 sensor power |
| Potentiometer 2 (J2) | Output / Wiper | ADC1 (GP27) | Green | Joint 2 sensor feedback |
| Potentiometer 3 (J3) | VCC / GND | 3V3 / GND | Red / Black | J3 sensor power |
| Potentiometer 3 (J3) | Output / Wiper | ADC2 (GP28) | Blue | Joint 3 sensor feedback |
| Push Button | Pin 1 | GPIO 16 | Orange | Capture signal pin |
| Push Button | Pin 2 | GND | Black | Ground reference |

> **Wiring tip:** The push button is configured with the internal pull-up resistor enabled. One side of the button connects directly to GPIO 16, and the opposite side connects to GND. When the button is pressed, it pulls the signal pin to GND (LOW).

## Code
```cpp
// Robotic Arm Position Memory Log - VEGA ARIES v3
const int POT_J1 = GP26;      // Joint 1 sensor on ADC0
const int POT_J2 = GP27;      // Joint 2 sensor on ADC1
const int POT_J3 = GP28;      // Joint 3 sensor on ADC2
const int BUTTON_PIN = 16;    // Capture button on GPIO 16

int prevButtonState = HIGH;   // Track previous button state (pull-up default is HIGH)
int recordCount = 0;          // Count of recorded coordinates

void setup() {
  // Configure the button pin with the internal pull-up resistor
  pinMode(BUTTON_PIN, INPUT_PULLUP);

  // Initialize Serial Monitor
  Serial.begin(9600);
  Serial.println("Teach-Mode Position Logger online. Set angles and press button.");
}

void loop() {
  // Read the current state of the button
  int buttonState = digitalRead(BUTTON_PIN);

  // Edge detection: Trigger only when the button goes from HIGH (released) to LOW (pressed)
  if (buttonState == LOW && prevButtonState == HIGH) {
    // Increment the record counter
    recordCount++;

    // Read the three joint sensors (0 - 4095)
    int valJ1 = analogRead(POT_J1);
    int valJ2 = analogRead(POT_J2);
    int valJ3 = analogRead(POT_J3);

    // Map raw sensor values to angles (0 - 180 degrees)
    int angleJ1 = map(valJ1, 0, 4095, 0, 180);
    int angleJ2 = map(valJ2, 0, 4095, 0, 180);
    int angleJ3 = map(valJ3, 0, 4095, 0, 180);

    // Log the coordinates to the Serial Monitor
    Serial.print("RECORD #");
    Serial.print(recordCount);
    Serial.print(" -> J1: ");
    Serial.print(angleJ1);
    Serial.print(" deg | J2: ");
    Serial.print(angleJ2);
    Serial.print(" deg | J3: ");
    Serial.print(angleJ3);
    Serial.println(" deg");

    // Small delay to debounce the mechanical contact bounce of the button press
    delay(200);
  }

  // Update previous button state for the next cycle comparison
  prevButtonState = buttonState;

  // Tiny delay to avoid running the ADC too fast and stabilize execution
  delay(10);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, three **Potentiometers**, and a **Push Button** onto the canvas.
2. Wire the Potentiometer outputs to **GP26 (ADC0)**, **GP27 (ADC1)**, and **GP28 (ADC2)**. Connect VCC to **3V3** and GND to **GND**.
3. Wire the Push Button: **Pin 1** to **GPIO 16**, and **Pin 2** to **GND**.
4. Paste the C++ code into the editor.
5. Select **Interpreted Mode** and click **Run**.
6. Move the three simulated potentiometer sliders to set different angles.
7. Click the virtual push button widget.
8. Check the Serial Monitor to see the recorded coordinates printed with a unique record number.

## Expected Output
Serial Monitor:
```
Teach-Mode Position Logger online. Set angles and press button.
RECORD #1 -> J1: 45 deg | J2: 90 deg | J3: 135 deg
RECORD #2 -> J1: 60 deg | J2: 120 deg | J3: 90 deg
RECORD #3 -> J1: 180 deg | J2: 15 deg | J3: 45 deg
```

## Expected Canvas Behavior
* Adjusting the three potentiometer sliders updates their internal positions.
* Clicking the button flashes the button widget state and prints a new coordinate line on the Serial Monitor.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(BUTTON_PIN, INPUT_PULLUP)` | Configures GPIO 16 as an input, engaging the internal pull-up resistor to pull the pin HIGH when the button is open. |
| `buttonState == LOW && prevButtonState == HIGH` | Evaluates if the button was just pressed (transitioning from HIGH to LOW). |
| `analogRead(POT_J1)` | Reads Joint 1 position sensor. |
| `map(valJ1, 0, 4095, 0, 180)` | Translates Joint 1 sensor voltage to degrees. |
| `prevButtonState = buttonState` | Records the button state to detect transitions in the next loop cycle. |

## Hardware & Safety Concept
* **Teach-Mode Robotics**: Industrial robots use "Teach Pendants" to manually guide the robot arm through a sequence of target points. By pressing a "Save" button, the controller logs the encoder values (angles) for all joints. These saved coordinates are stored in memory and replayed sequentially during playback mode.
* **Button Debouncing and State Changes**: Mechanical switches contain metal contacts that spring back and forth when pressed, creating brief voltage spikes (contact bounce) for up to 10-20 ms. Without a debounce delay or edge detection state machine, the micro-controller will register these spikes as multiple separate presses, saving duplicate coordinates on a single click.

## Try This! (Challenges)
1. **LED Save Indicator**: Connect an external LED to GPIO 15. Program it to blink ON for 150 ms whenever a coordinate record is successfully saved, providing physical feedback to the user.
2. **Calibration Test Mode**: Add logic so that if the button is held down for more than 2 seconds, it resets `recordCount` back to 0 and prints "Memory Cleared" to the Serial Monitor.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Clicking the button once saves multiple records | Missing debounce delay | Verify that `delay(200)` is included inside the button-press detection block. |
| The coordinates print continuously without pressing the button | Button wired incorrectly or wrong pull-up mode | Ensure the button pin is set to `INPUT_PULLUP`. Verify that the button is wired between GPIO 16 and GND. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [16 - Push Button LED Toggle](../beginner/16-push-button-led-toggle.md)
- [134 - 3-Axis Robotic Arm Claw Controller](134-3-axis-robotic-arm-claw-controller.md)
