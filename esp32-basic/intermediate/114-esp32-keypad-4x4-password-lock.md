# 114 - ESP32 Keypad 4x4 Password Lock

Build an electronic combination lock that accepts a passcode from a 4x4 matrix keypad and actuate a servo motor to release a deadbolt.

## Goal
Learn how to scan matrix keypads using the `Keypad` library, collect character inputs into password arrays, verify matches, and drive lock servos.

## What You Will Build
A 4x4 matrix keypad is connected to 8 GPIO pins. A servo motor on GPIO 27 acts as the physical locking bolt. The user must enter a correct 4-digit passcode (e.g. "1379"). Pressing '#' submits the passcode, rotating the servo to 90° (unlocked) for 3 seconds before returning to 0° (locked). Pressing '*' clears the entry buffer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Keypad Row 1 | Pin 1 | GPIO12 | Red | Keypad Row outputs |
| Keypad Row 2 | Pin 2 | GPIO13 | Orange | Keypad Row outputs |
| Keypad Row 3 | Pin 3 | GPIO14 | Yellow | Keypad Row outputs |
| Keypad Row 4 | Pin 4 | GPIO15 | Green | Keypad Row outputs |
| Keypad Col 1 | Pin 5 | GPIO16 | Blue | Keypad Column inputs |
| Keypad Col 2 | Pin 6 | GPIO17 | Purple | Keypad Column inputs |
| Keypad Col 3 | Pin 7 | GPIO25 | White | Keypad Column inputs |
| Keypad Col 4 | Pin 8 | GPIO26 | Gray | Keypad Column inputs |
| Servo Motor | Signal | GPIO27 | Brown | Lock arm PWM control |
| Servo Motor | VCC / GND | 5V / GND | Red / Black | Power rails |

> **Wiring tip:** The 4x4 matrix keypad uses 8 wires. Connect rows to GPIO 12, 13, 14, and 15, and columns to GPIO 16, 17, 25, and 26. Wire the servo signal to **GPIO 27**.

## Code
```cpp
// Keypad 4x4 Password Lock
#include <Keypad.h>
#include <ESP32Servo.h>

const int SERVO_PIN = 27;

// Keypad Configuration
const byte ROWS = 4;
const byte COLS = 4;

char keys[ROWS][COLS] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};

byte rowPins[ROWS] = {12, 13, 14, 15}; // Connect to Row Pinouts
byte colPins[COLS] = {16, 17, 25, 26}; // Connect to Col Pinouts

Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);
Servo lockServo;

// Secret passcode definition
const String SECRET_CODE = "1379";
String inputCode = "";

void setup() {
  Serial.begin(115200);
  
  lockServo.attach(SERVO_PIN);
  lockServo.write(0); // Start locked
  
  Serial.println("Keypad Security Lock ready. Enter Passcode:");
}

void loop() {
  char key = keypad.getKey();
  
  if (key) {
    Serial.print("Pressed: "); Serial.println(key);
    
    // '#' acts as the SUBMIT key
    if (key == '#') {
      if (inputCode == SECRET_CODE) {
        Serial.println(">> ACCESS GRANTED <<");
        lockServo.write(90); // Rotate to unlock
        delay(3000);         // Keep unlocked for 3s
        lockServo.write(0);  // Relock
        Serial.println("Relocked. Enter Passcode:");
      } else {
        Serial.println(">> ACCESS DENIED: Invalid Passcode <<");
        // Shake the servo slightly to indicate error
        lockServo.write(10); delay(150); lockServo.write(0);
      }
      inputCode = ""; // Clear buffer
    } 
    // '*' acts as the CLEAR key
    else if (key == '*') {
      Serial.println("Entry cleared.");
      inputCode = "";
    } 
    // Append characters up to 4 digits
    else {
      if (inputCode.length() < 4) {
        inputCode += key;
        Serial.print("Current: "); Serial.println(inputCode);
      }
    }
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **4x4 Keypad**, and **Servo** onto the canvas.
2. Wire the 8 Keypad pins and Servo signal to **GPIO27** as listed in the table.
3. Paste the code and click **Run**.
4. Click numbers `1`, `3`, `7`, `9` on the keypad widget and then click `#`. Watch the servo sweep 90 degrees.

## Expected Output
Serial Monitor:
```
Keypad Security Lock ready. Enter Passcode:
Pressed: 1
Current: 1
Pressed: 3
Current: 13
Pressed: 7
Pressed: 9
Pressed: #
>> ACCESS GRANTED <<
```

## Expected Canvas Behavior
* Pressing keys on the virtual keypad widget outputs character logs.
* Entering `1379#` sweeps the servo widget to 90° for 3 seconds before returning to 0°.
* Entering incorrect codes wiggles the servo briefly to simulate a locked handle.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `keypad.getKey()` | Scans the row and column matrix pins to detect if a key contact is closed. |
| `inputCode == SECRET_CODE` | Compares the entered string against the target passcode. |
| `lockServo.write(90)` | Unlocks the door latch. |

## Hardware & Safety Concept: Matrix Scanning Logic
A matrix keypad uses a grid of rows and columns to reduce the number of pins required. If we connected 16 buttons individually, we would need 16 GPIO pins. In a 4x4 matrix, we only need 8 pins (4 rows, 4 columns). The library drives one row LOW at a time and scans the columns to detect if a button connection is closed, allowing fast detection without pin conflicts.

## Try This! (Challenges)
1. **Interactive Status LCD**: Add a 16x2 I2C LCD (Project 58) showing "Enter Pin:" and masking character entries with asterisks ("****").
2. **Alert Buzzer**: Add a buzzer on GPIO 4 that plays a chirp for key presses and a warning siren if the password fails.
3. **Change Password Mode**: Add a routine allowing the user to set a new password if they enter the current master password followed by 'A'.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Key presses are ignored or incorrect | Rows and Columns swapped | Verify row and column pin arrays match the physical wiring |
| Multiple keys trigger on a single press | Floating inputs | Ensure pins are securely connected; the library handles input pull-ups automatically |
| Servo does not lock back | Program halted in delay | Confirm power rails are connected to 5V Vin |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [115 - ESP32 Keypad Display HUD](115-esp32-keypad-display-hud.md)
- [87 - ESP32 MFRC522 RFID Servo Door Unlock](87-esp32-mfrc522-rfid-servo-door-unlock.md)
- [51 - ESP32 Servo Motor 180° Sweep](51-esp32-servo-motor-180-sweep.md)
