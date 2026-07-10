# 90 - ESP32 Bluetooth LED Controller (String Parsing)

Control a physical LED wirelessly by sending text commands ("RED_ON", "RED_OFF") from a Bluetooth terminal app to the ESP32.

## Goal
Learn how to read serial data into a string buffer, parse text commands using string comparison operations, and switch outputs based on the received commands.

## What You Will Build
An HC-05 Bluetooth module on UART2 (RX2: GPIO 16, TX2: GPIO 17) receives serial commands. A Red LED is connected to GPIO 5. Sending "RED_ON" turns the LED on; "RED_OFF" turns it off.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth_hc05` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-05 Module | VCC | 5V (Vin) | Red | Power |
| HC-05 Module | GND | GND | Black | Ground |
| HC-05 Module | TXD | GPIO16 (RX2) | Green | Serial Data |
| HC-05 Module | RXD | GPIO17 (TX2) | Yellow | Serial Data |
| LED (Red) | Anode (+) | GPIO5 via 330 Ω | Orange | Indicator light |
| LED (Red) | Cathode (−) | GND | Black | Ground reference |

> **Wiring tip:** Connect the LED anode to **GPIO 5** using a 330 Ω resistor to protect the pin from overcurrent. The Bluetooth module wiring is identical to Project 89.

## Code
```cpp
// HC-05 Bluetooth LED Controller (String Parsing)
#define RX2_PIN 16
#define TX2_PIN 17

const int LED_PIN = 5;
String inputString = ""; // String buffer to hold incoming serial data

void setup() {
  Serial.begin(115200);
  
  // HC-05 Default Baud Rate
  Serial2.begin(9600, SERIAL_8N1, RX2_PIN, TX2_PIN);
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW); // Start with LED OFF
  
  Serial.println("Bluetooth LED Controller Active. Send command...");
}

void loop() {
  // Read incoming characters from Bluetooth
  while (Serial2.available() > 0) {
    char inChar = (char)Serial2.read();
    
    // Check for newline or carriage return as message terminator
    if (inChar == '\n' || inChar == '\r') {
      inputString.trim(); // Remove leading/trailing whitespaces or carriage returns
      
      if (inputString.length() > 0) {
        processCommand(inputString);
      }
      
      inputString = ""; // Clear buffer
    } else {
      inputString += inChar; // Append character to buffer
    }
  }
}

void processCommand(String cmd) {
  Serial.print("Received Command: ");
  Serial.println(cmd);
  
  if (cmd.equalsIgnoreCase("RED_ON")) {
    digitalWrite(LED_PIN, HIGH);
    Serial2.println("SUCCESS: Red LED is now ON");
    Serial.println("Action: LED ON");
  } 
  else if (cmd.equalsIgnoreCase("RED_OFF")) {
    digitalWrite(LED_PIN, LOW);
    Serial2.println("SUCCESS: Red LED is now OFF");
    Serial.println("Action: LED OFF");
  } 
  else {
    Serial2.println("ERROR: Unknown Command. Use RED_ON or RED_OFF");
    Serial.println("Action: Unknown");
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **HC-05 Bluetooth Module**, and **LED** onto the canvas.
2. Wire HC-05 to **GPIO16/GPIO17**. Wire LED to **GPIO5**.
3. Paste the code and click **Run**.
4. In the virtual Bluetooth input text console, type `RED_ON` and press Enter. Watch the LED light up.

## Expected Output
Serial Monitor:
```
Bluetooth LED Controller Active. Send command...
Received Command: RED_ON
Action: LED ON
Received Command: RED_OFF
Action: LED OFF
Received Command: HELLO
Action: Unknown
```

Bluetooth Terminal App:
```
SUCCESS: Red LED is now ON
SUCCESS: Red LED is now OFF
ERROR: Unknown Command. Use RED_ON or RED_OFF
```

## Expected Canvas Behavior
* Sending `RED_ON\n` from the simulated Bluetooth console turns on the LED widget.
* Sending `RED_OFF\n` turns off the LED widget.
* The system responds with status strings over the Bluetooth link.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `inChar == '\n' \|\| inChar == '\r'` | Detects message termination (Enter key press) to trigger parsing. |
| `inputString.trim()` | Clears whitespace characters like `\r` to ensure string comparison matches exactly. |
| `cmd.equalsIgnoreCase("RED_ON")` | Compares strings without case sensitivity (accepts "red_on", "RED_ON", "Red_On"). |
| `Serial2.println(...)` | Sends confirmation messages back to the mobile phone/terminal app. |

## Hardware & Safety Concept: Command Parsing Security
Command parsers should include safety fallbacks. If a garbage or corrupted string is received, the code should discard it and do nothing rather than triggering unexpected behavior. Using case-insensitive checks (`equalsIgnoreCase`) reduces user errors during manual operations.

## Try This! (Challenges)
1. **Toggle Command**: Add a command "TOGGLE" that switches the current state of the LED.
2. **RGB Controller**: Connect Red, Green, and Blue LEDs (or an RGB module) and write parsing rules for "GREEN_ON", "BLUE_OFF", etc.
3. **PWM Dimmer Command**: Parse speed/intensity values (e.g. "DIM:128" or "DIM:255") using substring split functions to set the LED brightness.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Commands are ignored | Missing newline terminators | Ensure your Bluetooth terminal app is configured to send `\n` or `\r` (LF/CR) at the end of strings |
| Commands match intermittently | Carriage return characters left in string | Ensure `inputString.trim()` is called before processing |
| LED does not light up | Bad pin wiring | Verify LED anode is wired to GPIO 5, cathode to GND |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [89 - ESP32 HC-05 Bluetooth Serial UART Link](89-esp32-hc05-bluetooth-serial-uart-link.md)
- [91 - ESP32 Bluetooth Buzzer Tone Player](91-esp32-hc05-bluetooth-buzzer-tone-player.md)
- [92 - ESP32 Bluetooth Relay Switcher](92-esp32-hc05-bluetooth-relay-switcher.md)
