# 75 - BT Relay Switch

Toggle an electrical relay module remotely by sending Bluetooth commands from a smartphone.

## Goal
Learn how to parse serial stream commands to toggle a relay module, isolating high-power circuits from the microcontroller.

## What You Will Build
The Arduino listens for Bluetooth serial commands:
- Sending `'O'` (Open/On) triggers the relay D4 ON, closing its output contacts.
- Sending `'F'` (Off) triggers the relay D4 OFF, opening its output contacts.

**Why D2, D3, and D4?** Pins D2/D3 manage the Bluetooth communication. Pin D4 controls the relay trigger.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-05 Bluetooth Module | `hc05_bluetooth` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-05 Module | TXD | D2 | Software RX |
| HC-05 Module | RXD | D3 | Software TX (Use divider on hardware) |
| HC-05 Module | VCC | 5V | Power supply |
| HC-05 Module | GND | GND | Ground reference |
| Relay Module | IN | D4 | Control pin |
| Relay Module | VCC | 5V | Relay coil power |
| Relay Module | GND | GND | Ground reference |

## Code
```cpp
#include <SoftwareSerial.h>

SoftwareSerial BT(2, 3); // RX = D2, TX = D3

const int RELAY_PIN = 4;

void setup() {
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start with relay OFF
  
  Serial.begin(9600);
  BT.begin(9600);
  
  Serial.println("Bluetooth Relay Switch Ready");
}

void loop() {
  if (BT.available() > 0) {
    char command = BT.read();
    
    Serial.print("Received: ");
    Serial.println(command);
    
    // Process commands
    if (command == 'O' || command == 'o') {
      digitalWrite(RELAY_PIN, HIGH);
      Serial.println("[ACTION] Relay triggered ON");
      BT.println("Relay is ON");
    } 
    else if (command == 'F' || command == 'f') {
      digitalWrite(RELAY_PIN, LOW);
      Serial.println("[ACTION] Relay triggered OFF");
      BT.println("Relay is OFF");
    }
  }
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **HC-05 Bluetooth Module**, and **Relay Module** onto the canvas.
2. Connect HC-05: **TXD** to **D2**, **RXD** to **D3**, **VCC** to **5V**, and **GND** to **GND**.
3. Connect Relay: **IN** to **D4**, **VCC** to **5V**, and **GND** to **GND**.
4. Paste the code into the editor.
5. Select the **Compiled Arduino Uno** runtime.
6. Click **Build and Run** (or **Run**).
7. Open the **Terminal** tab in the bottom dock.
8. Send characters `'O'` and `'F'` over the simulated Bluetooth port to switch the relay state.

## Expected Output

Terminal:
```
Bluetooth Relay Switch Ready
Received: O
[ACTION] Relay triggered ON
Received: F
[ACTION] Relay triggered OFF
...
```
The relay contacts switch closed and open on the canvas as commands are received.

## Expected Canvas Behavior

| Sent Bluetooth Command | Pin D4 Output | Relay State | Status Return over BT |
| --- | --- | --- | --- |
| 'O' | HIGH (5V) | Contacts Closed (ON) | "Relay is ON" |
| 'F' | LOW (0V) | Contacts Open (OFF) | "Relay is OFF" |

The relay indicator state changes instantly when the command characters match.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `digitalWrite(RELAY_PIN, HIGH)` | Drives the output pin HIGH, turning on the switching transistor on the relay module. |

## Hardware & Safety Concept: Electrical Isolation
Relays provide physical **galvanic isolation** between control circuits and power circuits. 
- The low-voltage side (Arduino) only powers an internal electromagnetic coil.
- The high-voltage side (connected to the relay terminals) is physically separated by an air gap. When the coil is energized, the magnetic field pulls a flexible metal contact to close the circuit.
This keeps high voltage (like 110V/220V AC mains) safely away from the sensitive low-voltage microchip.

## Try This! (Challenges)
1. **Timed Pulse**: Add a command `'P'` (Pulse) that turns the relay ON for 1 second, then automatically turns it back OFF.
2. **Toggle Logic**: Modify the code so that sending a single character `'T'` toggles the relay state (if ON -> OFF, if OFF -> ON).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay does not switch | Incorrect runtime selected | Verify that you have selected the **Compiled Arduino Uno** runtime. |
| Relay clicks but no power flows | Load wiring error | Ensure your external load is wired in series through the Relay's **COM** and **NO** terminals. |

## Mode Notes
This project **requires the Compiled Arduino Uno** runtime because custom byte parsing logic is not processed by the interpreted mode runner.

## Related Projects
- [39 - Auto Lamp Relay](39-auto-lamp-relay.md)
- [73 - BT LED Control](73-bt-led-control.md)
- [74 - BT Buzzer Trigger](74-bt-buzzer-trigger.md)
