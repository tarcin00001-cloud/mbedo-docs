# 114 - Keypad 4x4 Password Lock

Build an electronic combination door lock using a 4x4 matrix keypad and a servo motor on the VEGA ARIES v3 board.

## Goal
Learn how to scan row-column switch matrix keypads, compile single character inputs into entry strings, and actuate locking servo mechanisms under interpreted mode rules.

## What You Will Build
An electronic security deadbolt. The system accepts inputs from a 4x4 keypad. When the user inputs the correct passcode ("1379") and presses `#` to submit, the locking servo rotates to 90 degrees (unlocked) for 3 seconds before automatically returning to 0 degrees (locked). Pressing `*` resets the input buffer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Keypad | Row 1 (Pin 1) | GPIO 2 | Red | Keypad Row 1 scan pin |
| Keypad | Row 2 (Pin 2) | GPIO 3 | Orange | Keypad Row 2 scan pin |
| Keypad | Row 3 (Pin 3) | GPIO 4 | Yellow | Keypad Row 3 scan pin |
| Keypad | Row 4 (Pin 4) | GPIO 5 | Green | Keypad Row 4 scan pin |
| Keypad | Col 1 (Pin 5) | GPIO 6 | Blue | Keypad Column 1 scan pin |
| Keypad | Col 2 (Pin 6) | GPIO 7 | Purple | Keypad Column 2 scan pin |
| Keypad | Col 3 (Pin 7) | GPIO 8 | White | Keypad Column 3 scan pin |
| Keypad | Col 4 (Pin 8) | GPIO 9 | Gray | Keypad Column 4 scan pin |
| Servo Motor | Signal | GPIO 13 | Yellow | Servo PWM control line |
| Servo Motor | VCC | 5V | Red | Servo power supply |
| Servo Motor | GND | GND | Black | Ground reference |

> **Wiring tip:** The 4x4 keypad uses 8 digital lines in total. Connect the first four pins (Rows 1-4) to GPIOs 2 to 5, and the next four pins (Columns 1-4) to GPIOs 6 to 9. Wire the servo signal pin to GPIO 13.

## Code
```cpp
#include <Keypad.h>
#include <Servo.h>

const int SERVO_PIN = 13; // Servo on GPIO 13

// Matrix configuration
const byte ROWS = 4;
const byte COLS = 4;

char keys[ROWS][COLS] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};

byte rowPins[ROWS] = {2, 3, 4, 5};   // Keypad row pins
byte colPins[COLS] = {6, 7, 8, 9};   // Keypad column pins

Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);
Servo lockServo;

const String SECRET_CODE = "1379";
String inputCode = "";

void setup() {
  lockServo.attach(SERVO_PIN);
  lockServo.write(0); // Initialize in locked state (0 degrees)
}

void loop() {
  char key = keypad.getKey();

  if (key) {
    // '#' triggers passcode submission
    if (key == '#') {
      if (inputCode == SECRET_CODE) {
        lockServo.write(90); // Rotate servo to open latch
        delay(3000);         // Keep unlocked for 3 seconds
        lockServo.write(0);  // Close latch (relock)
      } else {
        // Access Denied: wiggle the servo slightly to show error
        lockServo.write(15); 
        delay(150); 
        lockServo.write(0);
      }
      inputCode = ""; // Reset entry buffer
    } 
    // '*' clears the current password buffer
    else if (key == '*') {
      inputCode = "";
    } 
    // Collect keypress character
    else {
      if (inputCode.length() < 4) {
        inputCode += key;
      }
    }
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **4x4 Matrix Keypad**, and **Servo Motor** components onto the canvas.
2. Wire the Keypad: Pins **1-4** to **GPIO 2-5**, and Pins **5-8** to **GPIO 6-9**.
3. Wire the Servo: **Signal** to **GPIO 13**, **VCC** to **5V**, and **GND** to **GND**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Enter Passcode:
Key Pressed: 1
Key Pressed: 3
Key Pressed: 7
Key Pressed: 9
Key Pressed: #
>> ACCESS GRANTED <<
```

## Expected Canvas Behavior
* Pressing keys `1`, `3`, `7`, `9` on the virtual keypad followed by `#` causes the servo to rotate to 90 degrees.
* After 3 seconds, the servo automatically rotates back to 0 degrees.
* Pressing incorrect codes and pressing `#` wiggles the servo arm slightly.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Keypad keypad = Keypad(...)` | Instantiates the Keypad mapping with row/column configurations. |
| `keypad.getKey()` | Polls rows and scans column logic states to identify closed key switches. |
| `inputCode == SECRET_CODE` | Compares the compiled user inputs against the target code string. |
| `lockServo.write(90)` | Sends the target PWM pulse width to rotate the lock arm. |

## Hardware & Safety Concept
* **Pin Multiplexing savings**: Standard matrix keypads arrange switches in rows and columns. Driving a row pin LOW allows scanning column states to identify which key is pressed. This reduces the pin requirement from 16 pins to only 8 pins, freeing up pins for other peripherals.
* **Hobby Servo Torque**: Ensure the physical deadbolt mechanism is aligned so the servo does not experience binding stress when locked. Continuous mechanical resistance will strip gears and burn out the motor.

## Try This! (Challenges)
1. **Access Indicator LED**: Connect a warning LED on GPIO 15. Flash the LED slowly while locked, light it green/HIGH when unlocked, and blink it rapidly if the password entry fails.
2. **Lockout Mechanism**: Track wrong entries. If the user enters a wrong passcode three times, disable passcode entry and block inputs for 10 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Keypad inputs are swapped or wrong keys print | Mismatched Row/Col wiring order | Trace the keypad ribbon. Ensure Row pins connect to 2–5 and Col pins connect to 6–9 in sequential order. |
| Servo stutters or doesn't move 90 degrees | Low current power supply | Powered servos from a high-quality USB power source, sharing ground with the board. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [51 - Servo Motor 180 Degree Sweep](51-servo-motor-180-degree-sweep.md)
- [115 - Keypad Display HUD](115-keypad-display-hud.md)
