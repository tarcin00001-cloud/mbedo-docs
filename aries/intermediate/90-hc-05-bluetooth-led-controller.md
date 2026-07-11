# 90 - HC-05 Bluetooth LED Controller

Control a physical LED wirelessly by sending text commands ("LED_ON", "LED_OFF") from a Bluetooth terminal app to the ARIES v3 board.

## Goal
Learn how to read serial data into a string buffer, parse text commands using string comparison operations, and switch outputs based on the received commands without using loops inside C++ code.

## What You Will Build
An HC-05 Bluetooth module on UART2 (RX2: GPIO 16, TX2: GPIO 17) receives serial commands. A Warning LED is connected to GPIO 15 of the ARIES v3 board. Sending "LED_ON" turns the LED on; "LED_OFF" turns it off.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth_hc05` | Yes | Yes |
| LED (Warning) | `led` | Yes | Yes (on GPIO 15) |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-05 Module | VCC | 5V | Red | Power |
| HC-05 Module | GND | GND | Black | Ground |
| HC-05 Module | TXD | GPIO 16 (RX2) | Green | Serial Data |
| HC-05 Module | RXD | GPIO 17 (TX2) | Yellow | Serial Data |
| Warning LED | Anode (+) | GPIO 15 | Orange | Indicator light |
| Warning LED | Cathode (−) | GND | Black | Ground reference |

> **Wiring tip:** Connect the LED anode to **GPIO 15** using a 330 Ω resistor to protect the pin from overcurrent. The Bluetooth module wiring is identical to Project 89.

## Code
```cpp
// HC-05 Bluetooth LED Controller
const int LED_PIN = 15;
String inputString = ""; // String buffer to hold incoming serial data

void setup() {
  Serial.begin(115200);
  Serial2.begin(9600);
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW); // Start with LED OFF
  
  Serial.println("Bluetooth LED Controller Active. Send command...");
}

void loop() {
  // Read incoming characters from Bluetooth
  if (Serial2.available() > 0) {
    char inChar = (char)Serial2.read();
    
    // Check for newline or carriage return as message terminator
    if (inChar == '\n' || inChar == '\r') {
      inputString.trim(); // Remove leading/trailing whitespaces
      
      if (inputString.length() > 0) {
        Serial.print("Received Command: ");
        Serial.println(inputString);
        
        if (inputString.equalsIgnoreCase("LED_ON")) {
          digitalWrite(LED_PIN, HIGH);
          Serial2.println("SUCCESS: LED is now ON");
          Serial.println("Action: LED ON");
        } 
        else if (inputString.equalsIgnoreCase("LED_OFF")) {
          digitalWrite(LED_PIN, LOW);
          Serial2.println("SUCCESS: LED is now OFF");
          Serial.println("Action: LED OFF");
        } 
        else {
          Serial2.println("ERROR: Unknown Command. Use LED_ON or LED_OFF");
          Serial.println("Action: Unknown");
        }
      }
      
      inputString = ""; // Clear buffer
    } else {
      inputString += inChar; // Append character to buffer
    }
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **HC-05 Bluetooth Module**, and **LED** onto the canvas.
2. Wire HC-05 to **GPIO 16/GPIO 17**. Wire LED to **GPIO 15**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.
6. In the virtual Bluetooth input text console, type `LED_ON` and press Enter. Watch the LED light up.

## Expected Output
Serial Monitor:
```
Bluetooth LED Controller Active. Send command...
Received Command: LED_ON
Action: LED ON
Received Command: LED_OFF
Action: LED OFF
```

Bluetooth Terminal App:
```
SUCCESS: LED is now ON
SUCCESS: LED is now OFF
```

## Expected Canvas Behavior
* Sending `LED_ON\n` from the simulated Bluetooth console turns on the LED widget.
* Sending `LED_OFF\n` turns off the LED widget.
* The system responds with status strings over the Bluetooth link.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `inChar == '\n' \|\| inChar == '\r'` | Detects message termination (Enter key press) to trigger parsing. |
| `inputString.trim()` | Clears whitespace characters like `\r` to ensure string comparison matches exactly. |
| `inputString.equalsIgnoreCase(...)` | Compares strings without case sensitivity (accepts "led_on", "LED_ON", "Led_On"). |
| `Serial2.println(...)` | Sends confirmation messages back to the mobile phone/terminal app. |

## Hardware & Safety Concept
* **Command Parsing Security**: Command parsers should include safety fallbacks. If a garbage or corrupted string is received, the code should discard it and do nothing rather than triggering unexpected behavior. Using case-insensitive checks (`equalsIgnoreCase`) reduces user errors during manual operations.

## Try This! (Challenges)
1. **Toggle Command**: Add a command "TOGGLE" that switches the current state of the LED.
2. **RGB Controller**: Connect an RGB module (Red on GP23, Green on GP24, Blue on GP25) and write parsing rules for "GREEN_ON", "BLUE_ON", etc.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Commands are ignored | Missing newline terminators | Ensure your Bluetooth terminal app is configured to send `\n` or `\r` (LF/CR) at the end of strings. |
| Commands match intermittently | Carriage return characters left in string | Ensure `inputString.trim()` is called before processing. |
| LED does not light up | Bad pin wiring | Verify LED anode is wired to GPIO 15, cathode to GND. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [89 - HC-05 Bluetooth Serial UART Link](89-hc-05-bluetooth-serial-uart.md)
- [91 - HC-05 Bluetooth Buzzer Tone](91-hc-05-bluetooth-buzzer-tone.md)
- [92 - HC-05 Bluetooth Relay Switcher](92-hc-05-bluetooth-relay-switcher.md)
