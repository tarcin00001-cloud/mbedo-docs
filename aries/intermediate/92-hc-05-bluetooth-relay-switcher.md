# 92 - HC-05 Bluetooth Relay Switcher

Control high-power appliances wirelessly by toggling a relay module using text commands sent from a Bluetooth terminal to the ARIES v3 board.

## Goal
Learn how to implement wireless control interfaces for high-voltage switching loads, parsing command strings to drive relay coils safely without using loops inside C++ code.

## What You Will Build
An HC-05 Bluetooth module on UART2 (pins 16, 17) receives commands. A relay module is connected to GPIO 15. Sending the command "RELAY_ON" closes the relay contacts (powering the connected appliance); "RELAY_OFF" opens the contacts.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth_hc05` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes (on GPIO 15) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-05 Module | VCC | 5V | Red | Power |
| HC-05 Module | GND | GND | Black | Ground |
| HC-05 Module | TXD | GPIO 16 (RX2) | Green | Serial Data |
| HC-05 Module | RXD | GPIO 17 (TX2) | Yellow | Serial Data |
| Relay Module | VCC | 5V | Red | Relay Coil Power |
| Relay Module | IN (Signal) | GPIO 15 | Orange | Relay Control Pin |
| Relay Module | GND | GND | Black | Ground reference |

> **Wiring tip:** Standard relay modules require 5V on VCC to energize the mechanical switching coils. Connect VCC to the ARIES 5V pin. The signal pin is wired to GPIO 15.

## Code
```cpp
// HC-05 Bluetooth Relay Switcher
const int RELAY_PIN = 15;
String inputString = ""; // String buffer to hold incoming serial data

void setup() {
  Serial.begin(115200);
  Serial2.begin(9600);
  
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start with relay OFF
  
  Serial.println("Bluetooth Relay Switcher Active. Send Command...");
}

void loop() {
  if (Serial2.available() > 0) {
    char inChar = (char)Serial2.read();
    
    // Check for message terminator
    if (inChar == '\n' || inChar == '\r') {
      inputString.trim();
      
      if (inputString.length() > 0) {
        Serial.print("Received Command: ");
        Serial.println(inputString);
        
        if (inputString.equalsIgnoreCase("RELAY_ON")) {
          digitalWrite(RELAY_PIN, HIGH);
          Serial2.println("RELAY STATE: ON");
          Serial.println("Relay energized (ON)");
        } 
        else if (inputString.equalsIgnoreCase("RELAY_OFF")) {
          digitalWrite(RELAY_PIN, LOW);
          Serial2.println("RELAY STATE: OFF");
          Serial.println("Relay de-energized (OFF)");
        } 
        else {
          Serial2.println("ERROR: Command unrecognized. Send RELAY_ON or RELAY_OFF");
          Serial.println("Unrecognized action ignored.");
        }
      }
      
      inputString = ""; // Clear buffer
    } else {
      inputString += inChar;
    }
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **HC-05 Bluetooth Module**, and **Relay** onto the canvas.
2. Wire HC-05 TXD/RXD to **GPIO 16/GPIO 17**. Connect Relay input to **GPIO 15**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.
6. In the virtual Bluetooth text console, type `RELAY_ON` and press Enter. Watch the relay switch state on the canvas.

## Expected Output
Serial Monitor:
```
Bluetooth Relay Switcher Active. Send Command...
Received Command: RELAY_ON
Relay energized (ON)
Received Command: RELAY_OFF
Relay de-energized (OFF)
```

Bluetooth Terminal:
```
RELAY STATE: ON
RELAY STATE: OFF
```

## Expected Canvas Behavior
* Sending `RELAY_ON\n` from the simulated Bluetooth console closes the relay contacts (widget highlights in green).
* Sending `RELAY_OFF\n` opens the relay contacts (widget highlights in grey).
* The relay state is latched until overridden by another command.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `inputString.equalsIgnoreCase("RELAY_ON")` | Compares the buffer string ignoring case differences. |
| `digitalWrite(RELAY_PIN, HIGH)` | Drives GPIO 15 HIGH to trigger the relay. |
| `Serial2.println("RELAY STATE: ON")` | Sends a confirmation status message back over Bluetooth. |
| `inputString = ""` | Clears the buffer to prepare for the next incoming command. |

## Hardware & Safety Concept
* **Wireless Automation Load Safety**: When switching high-power loads (e.g. heaters, mains lamps) wirelessly, safety is critical. The system must start with the relay OFF on reboot or power loss (`digitalWrite(RELAY_PIN, LOW)` in `setup()`). If a communications error occurs, the relay should fail into a safe disconnected state.

## Try This! (Challenges)
1. **Interactive Status HUD**: Connect an I2C LCD (Project 58). Display the current relay power state ("LOAD STATE: ON/OFF") in real time on the LCD screen.
2. **Timed Pulse Command**: Add a command "PULSE" that turns the relay ON for exactly 5 seconds, then turns it OFF automatically.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay does not activate | VCC connected to 3.3V pin | Re-route the relay VCC connection to the 5V pin. |
| Commands are not recognized | Carriage return chars left in buffer | Ensure `inputString.trim()` is active before processing. |
| Bluetooth connection drops out | Unstable power supply | Ensure the ARIES board is powered from a high-quality USB connection. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [10 - Relay Module Switch](../beginner/10-relay-module-switch.md)
- [88 - MFRC522 RFID Relay Control](88-mfrc522-rfid-relay-control.md)
- [89 - HC-05 Bluetooth Serial UART Link](89-hc-05-bluetooth-serial-uart.md)
