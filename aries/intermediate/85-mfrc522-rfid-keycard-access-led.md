# 85 - MFRC522 RFID Keycard Access LED

Build a security access gate indicator that verifies RFID cards, lighting an external warning LED for authorized keys and executing a warning blink for denied attempts.

## Goal
Learn how to store authorization codes, compare scanned tag bytes against master keys, and drive visual status indicators without using loops inside C++ code.

## What You Will Build
An MFRC522 reader connected via SPI. The code holds a target 4-byte master key UID. If the scanned card matches the key, a Warning LED on GPIO 15 turns on for 2 seconds. Otherwise, the LED blinks rapidly to warn of access denial.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MFRC522 RFID Reader Module | `rfid_reader` | Yes | Yes |
| RFID Key Fob / Card (13.56MHz) | `rfid_tag` | Yes | Yes |
| LED (Warning) | `led` | Yes | Yes (on GPIO 15) |
| 330 Ω Resistor | `resistor` | No | Yes |

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
| Warning LED | Anode (+) | GPIO 15 | Green | Access status light |
| Warning LED | Cathode (−) | GND | Black | Ground reference |

> **Wiring tip:** Connect the Warning LED anode to GPIO 15 using a 330 Ω resistor to protect the pin from overcurrent. Share the common ground rail with the RFID module.

## Code
```cpp
// MFRC522 RFID Keycard Access LED
#include <SPI.h>
#include <MFRC522.h>

#define SS_PIN 10
#define RST_PIN 9

const int LED_PIN = 15;

MFRC522 rfid(SS_PIN, RST_PIN);

// Authorized Master Key UID bytes (Change this to match your tag)
const byte MASTER_0 = 0x83;
const byte MASTER_1 = 0xA2;
const byte MASTER_2 = 0xC3;
const byte MASTER_3 = 0xE4;

void setup() {
  Serial.begin(115200);
  SPI.begin();
  rfid.PCD_Init();
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW); // Start with LED OFF
  
  Serial.println("Access Control ready. Scan Card...");
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
    Serial.println(">> ACCESS GRANTED <<");
    digitalWrite(LED_PIN, HIGH);
    delay(2000); // Hold gate open indicator for 2 seconds
    digitalWrite(LED_PIN, LOW);
  } else {
    Serial.println(">> ACCESS DENIED <<");
    // Alarm blink sequence
    digitalWrite(LED_PIN, HIGH);
    delay(200);
    digitalWrite(LED_PIN, LOW);
    delay(200);
    digitalWrite(LED_PIN, HIGH);
    delay(200);
    digitalWrite(LED_PIN, LOW);
  }
  
  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **MFRC522 RFID Reader**, and **LED** onto the canvas.
2. Wire the MFRC522 SPI lines as listed. Connect the LED to **GPIO 15**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.
6. Drag a card tag to the reader widget. If its UID is `83 A2 C3 E4` the LED lights up for 2 seconds. If it differs, the LED flashes rapidly.

## Expected Output
Serial Monitor:
```
Access Control ready. Scan Card...
Scanned UID: 83 A2 C3 E4
>> ACCESS GRANTED <<
Scanned UID: 12 34 56 78
>> ACCESS DENIED <<
```

## Expected Canvas Behavior
* Scanning the matching card tag lights the LED widget on GPIO 15 for 2 seconds.
* Scanning any non-matching card tag flashes the LED widget rapidly.
* The LED stays OFF when no tag is active.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `MASTER_0` to `MASTER_3` | Defines the 4-byte master key values. |
| `rfid.uid.uidByte[0] == MASTER_0` | Directly evaluates if the first scanned byte matches the master key byte. |
| `digitalWrite(LED_PIN, HIGH)` | Drives GPIO 15 HIGH on successful scan. |
| `delay(200)` | Controls the blink rate for denied access. |

## Hardware & Safety Concept
* **Local Comparative Security**: Access control systems utilize local lookup arrays to check cards instantly without needing network connections. If a database is unreachable, local cached lists of master UIDs allow authorized personnel to unlock doors during power outages or connectivity losses.

## Try This! (Challenges)
1. **Access Counter**: Track the number of successful accesses and flash the LED that many times when an authorized card is scanned.
2. **Keycard Registration Mode**: Add a Push Button on GPIO 16. If pressed, store the next scanned tag's UID as the new master key.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| All scanned cards say ACCESS DENIED | Actual UID does not match the hardcoded values | Scan the card using Project 84 first, read the hex output, and copy those exact bytes into the MASTER constants. |
| LED does not light up | Polarity reversed or wrong pin | Verify that LED anode connects to GPIO 15. |
| Sensor doesn't read | Reset pin issue | Verify RST is connected to GPIO 9 and initialised. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [84 - MFRC522 RFID Card UID Serial Logs](84-mfrc522-rfid-card-uid-serial.md)
- [86 - MFRC522 RFID Keycard Access Buzzer](86-mfrc522-rfid-keycard-access-buzzer.md)
- [87 - MFRC522 RFID Servo Door Unlock](87-mfrc522-rfid-servo-door-unlock.md)
