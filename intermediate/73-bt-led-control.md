# 73 - BT LED Control

Control the onboard Arduino LED remotely by sending Bluetooth command signals from a smartphone.

## Goal
Learn how to read incoming serial characters from a Bluetooth stream, parse them as commands, and control digital output pins.

## What You Will Build
The Arduino listens on its virtual Bluetooth port. When a user sends the character `'1'` over Bluetooth, the LED connected to D13 turns ON. When they send the character `'0'`, the LED turns OFF.

**Why D2, D3, and D13?** Pins D2/D3 manage the Bluetooth communication. Pin D13 controls the warning/status LED.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-05 Bluetooth Module | `hc05_bluetooth` | Yes | Yes |
| LED | `led` | Optional (onboard D13) | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-05 Module | TXD | D2 | Software RX |
| HC-05 Module | RXD | D3 | Software TX (Use divider on hardware) |
| HC-05 Module | VCC | 5V | Power supply |
| HC-05 Module | GND | GND | Ground reference |
| LED | A | D13 | Control pin (onboard LED) |
| LED | C | GND | Ground reference |

## Code
```cpp
#include <SoftwareSerial.h>

SoftwareSerial BT(2, 3); // RX = D2, TX = D3

const int LED_PIN = 13;

void setup() {
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW); // Start with LED OFF
  
  Serial.begin(9600);
  BT.begin(9600);
  
  Serial.println("Bluetooth LED Controller Ready");
}

void loop() {
  // Check if any characters are waiting in the Bluetooth buffer
  if (BT.available() > 0) {
    char command = BT.read();
    
    Serial.print("Received: ");
    Serial.println(command);
    
    // Process commands
    if (command == '1') {
      digitalWrite(LED_PIN, HIGH);
      BT.println("LED turned ON");
    } 
    else if (command == '0') {
      digitalWrite(LED_PIN, LOW);
      BT.println("LED turned OFF");
    }
  }
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** and **HC-05 Bluetooth Module** onto the canvas.
2. Connect HC-05 **TXD** to Arduino **D2**, **RXD** to Arduino **D3**, **VCC** to Arduino **5V**, and **GND** to Arduino **GND**.
3. Paste the code into the editor.
4. Select the **Compiled Arduino Uno** runtime (Bluetooth logic requires full compiler support).
5. Click **Build and Run** (or **Run**).
6. Open the **Terminal** tab in the bottom dock.
7. Send characters `'1'` and `'0'` over the simulated Bluetooth port to toggle the LED.

## Expected Output

Terminal:
```
Bluetooth LED Controller Ready
Received: 1
Received: 0
...
```
The onboard LED (D13) on the canvas turns ON when `'1'` is received, and OFF when `'0'` is received.

## Expected Canvas Behavior

| Sent Bluetooth Command | Pin D13 State | LED State | Status Return over BT |
| --- | --- | --- | --- |
| '1' | HIGH (5V) | ON | "LED turned ON" |
| '0' | LOW (0V) | OFF | "LED turned OFF" |

The LED updates instantly upon command reception.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `char command = BT.read()` | Retrieves the oldest character from the incoming serial buffer queue. |
| `if (command == '1')` | Standard character comparison. If true, it changes the pin state. |
| `BT.println("...")` | Sends a text confirmation back over the Bluetooth link to the master device. |

## Hardware & Safety Concept: Wireless Remote Operation
Wireless control systems require clear communication checks.
- If a smartphone control app sends commands too fast, the serial buffer on the Arduino can overflow (which has a default limit of 64 bytes).
- In real-world robotic or home automation builds, it is common to append a **start and stop marker** (e.g. `[1]` or `<1>`) or use checksums to verify that the byte packet received was not corrupted during transmission.

## Try This! (Challenges)
1. **Toggle Switch**: Modify the code to use a single character command (e.g. `'T'`) to toggle the LED state on each press.
2. **Flash Command**: Add a new command `'F'` that makes the LED flash ON and OFF three times.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED does not respond to clicks | Incorrect runtime selected | Verify that you have selected the **Compiled Arduino Uno** runtime. Bluetooth parsing code is not executed in interpreted mode. |
| Command prints but LED is dead | Wrong pin used | Ensure your LED is wired to D13, matching `LED_PIN = 13`. |

## Mode Notes
This project **requires the Compiled Arduino Uno** runtime because custom byte parsing and nested comparison logic exceed the regex shims of the interpreted mode.

## Related Projects
- [01 - LED Blink](../beginner/01-led-blink.md)
- [72 - BT Serial Bridge](72-bt-serial-bridge.md)
- [74 - BT Buzzer Trigger](74-bt-buzzer-trigger.md)
