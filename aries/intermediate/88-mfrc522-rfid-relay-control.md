# 88 - MFRC522 RFID Relay Control

Build a wireless latching relay controller that toggles the state of a relay module (ON or OFF) each time an authorized RFID card is scanned.

## Goal
Learn how to use RFID scans to toggle persistent state variables, controlling external switching loads like home appliances safely without using loops inside C++ code.

## What You Will Build
An MFRC522 reader is connected via SPI. A Relay Module is connected to GPIO 15 of the ARIES v3 board. Scanning an authorized card switches the relay state (toggles it ON if it was OFF, and OFF if it was ON).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MFRC522 RFID Reader Module | `rfid_reader` | Yes | Yes |
| RFID Key Fob / Card (13.56MHz) | `rfid_tag` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes (on GPIO 15) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 Module | 3.3V | 3V3 | Red | Power |
| MFRC522 Module | GND | GND | Black | Ground |
| MFRC522 Module | RST (Reset) | GPIO 9 | Purple | Reset line |
| MFRC522 Module | SDA (SS) | GPIO 10 | Orange | Chip Select |
| MFRC522 Module | SCK | SPI SCK | Yellow | SPI Clock |
| MFRC522 Module | MISO | SPI MISO | Green | SPI MISO |
| MFRC522 Module | MOSI | SPI MOSI | Blue | SPI MOSI |
| Relay Module | VCC | 5V | Red | Relay Coil Power |
| Relay Module | IN (Signal) | GPIO 15 | Orange | Relay Control Pin |
| Relay Module | GND | GND | Black | Ground reference |

> **Wiring tip:** Standard relay modules require 5V on VCC to energize the internal switching coil. Connect the relay's VCC to the 5V rail on ARIES.

## Code
```cpp
// MFRC522 RFID Relay Control
#include <SPI.h>
#include <MFRC522.h>

#define SS_PIN 10
#define RST_PIN 9
const int RELAY_PIN = 15;

MFRC522 rfid(SS_PIN, RST_PIN);

// Authorized Master Key UID bytes (Change this to match your tag)
const byte MASTER_0 = 0x83;
const byte MASTER_1 = 0xA2;
const byte MASTER_2 = 0xC3;
const byte MASTER_3 = 0xE4;

int relayState = LOW;

void setup() {
  Serial.begin(115200);
  SPI.begin();
  rfid.PCD_Init();
  
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, relayState); // Start with relay OFF
  
  Serial.println("RFID Relay Control ready. Scan Card...");
}

void loop() {
  if ( ! rfid.PICC_IsNewCardPresent()) return;
  if ( ! rfid.PICC_ReadCardSerial()) return;
  
  Serial.print("Scanned UID: ");
  Serial.print(rfid.uid.uidByte[0], HEX);
  Serial.print(" ");
  Serial.print(rfid.uid.uidByte[1], HEX);
  Serial.print(" ");
  Serial.print(rfid.uid.uidByte[2], HEX);
  Serial.print(" ");
  Serial.println(rfid.uid.uidByte[3], HEX);
  
  // Compare scanned UID bytes with master key
  if (rfid.uid.uidByte[0] == MASTER_0 &&
      rfid.uid.uidByte[1] == MASTER_1 &&
      rfid.uid.uidByte[2] == MASTER_2 &&
      rfid.uid.uidByte[3] == MASTER_3) {
    Serial.println("Authorized! Toggling Relay...");
    
    // Toggle state
    if (relayState == LOW) {
      relayState = HIGH;
    } else {
      relayState = LOW;
    }
    
    digitalWrite(RELAY_PIN, relayState);
    delay(1500); // Simple debounce window to prevent rapid double-triggering
  } else {
    Serial.println("Unauthorized Card! Relay state unchanged.");
  }
  
  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **MFRC522 RFID Reader**, and **Relay** onto the canvas.
2. Wire the MFRC522 SPI lines as listed. Connect the Relay input to **GPIO 15**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.
6. Drag a card tag to the reader widget. If its UID is `83 A2 C3 E4` the Relay switches state. Repeat to toggle it off.

## Expected Output
Serial Monitor:
```
RFID Relay Control ready. Scan Card...
Scanned UID: 83 A2 C3 E4
Authorized! Toggling Relay...
Scanned UID: 83 A2 C3 E4
Authorized! Toggling Relay...
```

## Expected Canvas Behavior
* Scanning the matching card tag toggles the relay state. The virtual relay module widget highlights in green/active, then turns grey/inactive on the next valid scan.
* Scanning any non-matching card tag does not affect the relay state.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `int relayState = LOW` | Stores the current state of the relay (OFF at start). |
| `relayState = HIGH` | Updates the state tracking variable. |
| `digitalWrite(RELAY_PIN, relayState)` | Drives the control pin according to the toggled state. |
| `delay(1500)` | Prevents the card from being read multiple times in a single pass. |

## Hardware & Safety Concept
* **Relay Isolation and Flyback Protection**: Toggling heavy loads requires mechanical or solid-state relays. Relay coils require high current to trigger and behave like large inductors. They can generate reverse voltage spikes (flyback voltage) when turned off. Commercial relay modules contain optocouplers to isolate the microcontroller from high-voltage coils, and flyback diodes to clamp reverse voltage spikes.

## Try This! (Challenges)
1. **Latching LED Indicator**: Connect a Warning LED to GPIO 14. Light the LED when the relay is ON, and turn it OFF when the relay is OFF.
2. **Auto-off Timer Mode**: Modify the code so that scanning the card turns the relay ON, but it turns OFF automatically after 10 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay does not toggle | Pin VCC is connected to 3.3V | Connect the relay module VCC pin to 5V. |
| Relay clicks but load stays OFF | Load wiring error | Ensure the load circuit connects through the Common (COM) and Normally Open (NO) terminals. |
| Rapid repeated toggling | Delay value too short | Increase the debounce delay to 2000 ms to allow time to remove the card. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [84 - MFRC522 RFID Card UID Serial Logs](84-mfrc522-rfid-card-uid-serial.md)
- [10 - Relay Module Switch](../beginner/10-relay-module-switch.md)
- [88 - ESP32 MFRC522 RFID Relay Control](../esp32-basic/intermediate/88-esp32-mfrc522-rfid-relay-control.md)
