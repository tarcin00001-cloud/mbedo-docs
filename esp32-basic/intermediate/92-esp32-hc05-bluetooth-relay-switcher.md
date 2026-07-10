# 92 - ESP32 HC-05 Bluetooth Relay Switcher

Control high-power appliances wirelessly by toggling a relay module using text commands sent from a Bluetooth terminal.

## Goal
Learn how to implement wireless control interfaces for high-voltage switching loads, parsing command strings to drive relay coils safely.

## What You Will Build
An HC-05 Bluetooth module on UART2 (pins 16, 17) receives commands. A relay module is connected to GPIO 13. Sending the command "RELAY_ON" closes the relay contacts (powering the connected appliance); "RELAY_OFF" opens the contacts.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth_hc05` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-05 Module | VCC | 5V (Vin) | Red | Power |
| HC-05 Module | GND | GND | Black | Ground |
| HC-05 Module | TXD | GPIO16 (RX2) | Green | Serial Data |
| HC-05 Module | RXD | GPIO17 (TX2) | Yellow | Serial Data |
| Relay Module | VCC | 5V (Vin) | Red | Relay Coil Power |
| Relay Module | IN (Signal) | GPIO13 | Orange | Relay Control Pin |
| Relay Module | GND | GND | Black | Ground reference |

> **Wiring tip:** Standard relay modules require 5V on VCC to energise the mechanical switching coils. Connect VCC to the ESP32 Vin pin. The signal pin is wired to GPIO 13.

## Code
```cpp
// HC-05 Bluetooth Relay Switcher
#define RX2_PIN 16
#define TX2_PIN 17

const int RELAY_PIN = 13;
String inputString = ""; // String buffer

void setup() {
  Serial.begin(115200);
  Serial2.begin(9600, SERIAL_8N1, RX2_PIN, TX2_PIN);
  
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start with relay OFF
  
  Serial.println("Bluetooth Relay Switcher Active. Send Command...");
}

void loop() {
  while (Serial2.available() > 0) {
    char inChar = (char)Serial2.read();
    
    // Check for message terminator
    if (inChar == '\n' || inChar == '\r') {
      inputString.trim();
      
      if (inputString.length() > 0) {
        processRelayCommand(inputString);
      }
      
      inputString = ""; // Clear buffer
    } else {
      inputString += inChar;
    }
  }
}

void processRelayCommand(String cmd) {
  Serial.print("Received Command: ");
  Serial.println(cmd);
  
  if (cmd.equalsIgnoreCase("RELAY_ON")) {
    digitalWrite(RELAY_PIN, HIGH);
    Serial2.println("RELAY STATE: ON");
    Serial.println("Relay energized (ON)");
  } 
  else if (cmd.equalsIgnoreCase("RELAY_OFF")) {
    digitalWrite(RELAY_PIN, LOW);
    Serial2.println("RELAY STATE: OFF");
    Serial.println("Relay de-energized (OFF)");
  } 
  else {
    Serial2.println("ERROR: Command unrecognized. Send RELAY_ON or RELAY_OFF");
    Serial.println("Unrecognized action ignored.");
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **HC-05 Bluetooth Module**, and **Relay** onto the canvas.
2. Wire HC-05 TXD/RXD to **GPIO16/GPIO17**. Connect Relay input to **GPIO13**.
3. Paste the code and click **Run**.
4. In the virtual Bluetooth text console, type `RELAY_ON` and press Enter. Watch the relay switch state on the canvas.

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
ERROR: Command unrecognized. Send RELAY_ON or RELAY_OFF
```

## Expected Canvas Behavior
* Sending `RELAY_ON\n` from the simulated Bluetooth console opens/closes the relay contacts (widget turns green/active).
* Sending `RELAY_OFF\n` turns the relay widget off (grey/inactive).
* The relay state is maintained latching until overridden.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `cmd.equalsIgnoreCase("RELAY_ON")` | Compares the buffer string ignoring uppercase/lowercase differences. |
| `digitalWrite(RELAY_PIN, HIGH)` | Drives GPIO 13 HIGH to trigger the relay. |
| `Serial2.println("RELAY STATE: ON")` | Sends a confirmation status message back over Bluetooth. |

## Hardware & Safety Concept: Wireless Automation Load Safety
When switching high-power loads (e.g. heaters, mains lamps) wirelessly, safety is critical. The system must start with the relay OFF on reboot or power loss (`digitalWrite(RELAY_PIN, LOW)` in `setup()`). If a communications error occurs, the relay should fail into a safe disconnected state.

## Try This! (Challenges)
1. **Interactive Status HUD**: Integrate an I2C LCD (Project 58) that displays the current relay power state ("LOAD POWER: ON") in real time.
2. **Timed Pulse Command**: Add a command "PULSE" that turns the relay ON for exactly 5 seconds, then turns it OFF automatically.
3. **Emergency Disconnect**: Implement a local override push button on GPIO 4 that shuts down the relay immediately.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay does not activate | VCC connected to 3.3V pin | Re-route the relay VCC connection to the 5V/Vin pin |
| Commands are not recognized | Carriage return chars left in buffer | Ensure `inputString.trim()` is active before processing |
| Bluetooth module does not pair | Connection status pin issue | The STATE pin can be left floating, ensure key pairing mode is not enabled |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [11 - ESP32 Relay Module Control](../beginner/11-esp32-relay-module-control.md)
- [88 - ESP32 MFRC522 RFID Relay Control](88-esp32-mfrc522-rfid-relay-control.md)
- [89 - ESP32 HC-05 Bluetooth Serial UART Link](89-esp32-hc05-bluetooth-serial-uart-link.md)
