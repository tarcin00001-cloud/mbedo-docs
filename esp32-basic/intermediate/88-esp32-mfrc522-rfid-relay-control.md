# 88 - ESP32 MFRC522 RFID Relay Control

Build an RFID-enabled power switch that toggles a relay state between ON and OFF when an authorized security tag is scanned.

## Goal
Learn how to use RFID credentials to build latching power control switches, toggling high-voltage circuits using low-voltage logic.

## What You Will Build
An MFRC522 RFID reader connected via SPI. A relay module on GPIO 13. Scanning an authorized RFID card toggles the relay state (switching it ON if it was OFF, and OFF if it was ON), acting as a secure key-card power switch.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MFRC522 RFID Reader Module | `rfid_reader` | Yes | Yes |
| RFID Key Fob / Card (13.56MHz) | `rfid_tag` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 Module | 3.3V | 3V3 | Red | Power |
| MFRC522 Module | GND | GND | Black | Ground |
| MFRC522 Module | RST (Reset) | GPIO22 | Purple | Reset line |
| MFRC522 Module | SDA (SS) | GPIO5 | Orange | Chip Select |
| MFRC522 Module | SCK | GPIO18 | Yellow | SPI Clock |
| MFRC522 Module | MISO | GPIO19 | Green | SPI MISO |
| MFRC522 Module | MOSI | GPIO23 | Blue | SPI MOSI |
| Relay Module | VCC | 5V (Vin) | Red | Relay coil power |
| Relay Module | IN (Signal) | GPIO13 | Orange | Relay switch control |
| Relay Module | GND | GND | Black | Ground reference |

> **Wiring tip:** The relay control pin is wired to **GPIO 13**. Standard relay modules require 5V to energize their switching coils reliably, so power the relay VCC from the ESP32 Vin pin.

## Code
```cpp
// MFRC522 RFID Relay Control
#include <SPI.h>
#include <MFRC522.h>

#define SS_PIN 5
#define RST_PIN 22
const int RELAY_PIN = 13;

MFRC522 rfid(SS_PIN, RST_PIN);

// Authorised Master Key UID
const byte MASTER_UID[] = {0x83, 0xA2, 0xC3, 0xE4};

bool relayState = false; // Tracks current relay state (false = OFF, true = ON)

void setup() {
  Serial.begin(115200);
  SPI.begin();
  rfid.PCD_Init();
  
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start with relay OFF
  
  Serial.println("RFID Latching Relay ready. Scan Card...");
}

void loop() {
  if ( ! rfid.PICC_IsNewCardPresent()) return;
  if ( ! rfid.PICC_ReadCardSerial()) return;
  
  bool accessGranted = true;
  
  for (byte i = 0; i < rfid.uid.size; i++) {
    if (rfid.uid.uidByte[i] != MASTER_UID[i]) {
      accessGranted = false;
    }
  }
  
  if (accessGranted) {
    // Toggle the relay state
    relayState = !relayState;
    digitalWrite(RELAY_PIN, relayState ? HIGH : LOW);
    
    Serial.print("Access Granted! Relay toggled to: ");
    Serial.println(relayState ? "ON" : "OFF");
    
    // Play a short chirp to confirm toggle (if optional buzzer is added)
  } else {
    Serial.println("Access Denied! Relay state unchanged.");
  }
  
  delay(1500); // Prevent rapid double toggling
  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MFRC522 Reader**, and **Relay** onto the canvas.
2. Connect MFRC522 SPI lines. Connect the Relay signal pin to **GPIO13**.
3. Paste the code and click **Run**.
4. Drag a card tag to the reader widget. If it matches the Master UID, the relay toggles open/close on the canvas.

## Expected Output
Serial Monitor:
```
RFID Latching Relay ready. Scan Card...
Access Granted! Relay toggled to: ON
Access Granted! Relay toggled to: OFF
Access Denied! Relay state unchanged.
```

## Expected Canvas Behavior
* Scanning the matching card tag changes the relay widget state between ON (green/active) and OFF (grey/inactive).
* Scanning a non-matching card tag does nothing to the relay and outputs an Access Denied message on the Serial Monitor.
* The state latching is maintained between scans.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `relayState = !relayState` | Inverts the state variable (toggle logic). |
| `digitalWrite(RELAY_PIN, ...)` | Sets the GPIO 13 output pin HIGH or LOW based on the updated state. |
| `delay(1500)` | Inserts a debounce cool-down so a single card swipe doesn't toggle the relay twice. |

## Hardware & Safety Concept: Industrial Latching Switches and Load Control
In industrial workshops, dangerous tools (like table saws or CNC drills) use RFID latching relays. Authorised operators scan their badges to enable power to the machine console. The relay isolates the low-voltage microchip control lines from the high-voltage AC mains power supplying the motor.

## Try This! (Challenges)
1. **Status indicator LEDs**: Add a green LED that lights up when the relay is ON, and a red LED when the relay is OFF.
2. **Auto-Timeout Relay**: Turn the relay ON when authorized, but automatically turn it OFF after 10 seconds if the user forgets to scan it again.
3. **Emergency Override Switch**: Add a push button on GPIO 4 that immediately shuts off the relay (emergency stop) bypassing the RFID check.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay does not click | Relay module VCC connected to 3.3V | Connect relay VCC to 5V (Vin) to supply enough current to turn on the relay coil |
| Relay toggles twice on one scan | Cool-down delay is too short | Increase the debounce `delay()` at the end of the loop to 2000ms |
| Relay clicks but no power flows | External load wired incorrectly | Verify external load connections are wired through the Common (COM) and Normally Open (NO) terminals of the relay block |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [84 - ESP32 MFRC522 RFID Card UID Serial Logs](84-esp32-mfrc522-rfid-card-uid-serial-logs.md)
- [85 - ESP32 MFRC522 RFID Keycard Access LED](85-esp32-mfrc522-rfid-keycard-access-led.md)
- [11 - ESP32 Relay Module Control](../beginner/11-esp32-relay-module-control.md)
