# 85 - ESP32 MFRC522 RFID Keycard Access LED

Build a security access gate indicator that verifies RFID cards, lighting a green LED for authorised keys and a red LED for denied attempts.

## Goal
Learn how to store hardcoded authorisation codes, compare scanned tag arrays against master keys, and drive visual status indicators.

## What You Will Build
An MFRC522 reader connected via SPI. The code holds a target 4-byte master key UID. If the scanned card matches the key, a Green LED on GPIO 12 turns on for 2 seconds. Otherwise, a Red LED on GPIO 14 turns on for 2 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MFRC522 RFID Reader Module | `rfid_reader` | Yes | Yes |
| RFID Key Fob / Card (13.56MHz) | `rfid_tag` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 330 Ω Resistors (2) | `resistor` | No | Yes |

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
| Green LED | Anode (+) | GPIO12 via 330 Ω | Green | Access granted light |
| Green LED | Cathode (−) | GND | Black | Ground reference |
| Red LED | Anode (+) | GPIO14 via 330 Ω | Orange | Access denied light |
| Red LED | Cathode (−) | GND | Black | Ground reference |

> **Wiring tip:** Share the common breadboard ground rails for both LED cathodes. Connect the anode pins to GPIO 12 and 14 respectively.

## Code
```cpp
// MFRC522 RFID Keycard Access LED
#include <SPI.h>
#include <MFRC522.h>

#define SS_PIN 5
#define RST_PIN 22

const int GREEN_LED = 12;
const int RED_LED = 14;

MFRC522 rfid(SS_PIN, RST_PIN);

// Authorised Master Key UID (Change this to match your physical tag)
const byte MASTER_UID[] = {0x83, 0xA2, 0xC3, 0xE4};

void setup() {
  Serial.begin(115200);
  SPI.begin();
  rfid.PCD_Init();
  
  pinMode(GREEN_LED, OUTPUT);
  pinMode(RED_LED, OUTPUT);
  
  digitalWrite(GREEN_LED, LOW);
  digitalWrite(RED_LED, LOW);
  
  Serial.println("Access Control ready. Scan Card...");
}

void loop() {
  if ( ! rfid.PICC_IsNewCardPresent()) return;
  if ( ! rfid.PICC_ReadCardSerial()) return;
  
  Serial.print("Scanned UID: ");
  bool accessGranted = true;
  
  // Compare scanned UID bytes with master key
  for (byte i = 0; i < rfid.uid.size; i++) {
    Serial.print(rfid.uid.uidByte[i], HEX);
    Serial.print(" ");
    
    // Check if byte differs from Master Key
    if (rfid.uid.uidByte[i] != MASTER_UID[i]) {
      accessGranted = false;
    }
  }
  Serial.println();
  
  if (accessGranted) {
    Serial.println(">> ACCESS GRANTED <<");
    digitalWrite(GREEN_LED, HIGH);
    digitalWrite(RED_LED, LOW);
    delay(2000); // Hold gate open for 2 seconds
    digitalWrite(GREEN_LED, LOW);
  } else {
    Serial.println(">> ACCESS DENIED <<");
    digitalWrite(GREEN_LED, LOW);
    digitalWrite(RED_LED, HIGH);
    delay(2000); // Lockout warning for 2 seconds
    digitalWrite(RED_LED, LOW);
  }
  
  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MFRC522 Reader**, and two **LEDs** onto the canvas.
2. Wire the MFRC522 SPI lines as listed. Connect the Green LED to **GPIO12** and Red LED to **GPIO14**.
3. Paste the code and click **Run**.
4. Drag a card tag to the reader widget. If its UID is `83 A2 C3 E4` the Green LED lights. If it differs, the Red LED lights.

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
* Scanning the matching card tag lights the Green LED widget for 2 seconds.
* Scanning any non-matching card tag lights the Red LED widget for 2 seconds.
* Both LEDs stay OFF when no tag is active.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `MASTER_UID[] = {0x83,...}` | Stores the 4-byte hexadecimal master key array. |
| `rfid.uid.uidByte[i] != MASTER_UID[i]` | Performs byte-by-byte comparison of the scanned tag against the master key. |
| `digitalWrite(GREEN_LED, HIGH)` | Drives the Green LED HIGH for 2 seconds on success. |

## Hardware & Safety Concept: Comparative Security Loops
Access gates utilize local lookup arrays to check cards instantly without needing network connections. If a database is unreachable, local cached lists of master UIDs allow authorized personnel to open doors during power outages or connectivity losses.

## Try This! (Challenges)
1. **Multiple Authorized Keys**: Expand `MASTER_UID` into a two-dimensional array to store up to 3 authorized cards.
2. **Access Timeout alert**: Blink the Red LED rapidly instead of holding it solid to indicate a failed attempt.
3. **Serial Register Card**: Modify the code so that if a button is pressed, the next scanned card's UID becomes the new `MASTER_UID`.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| All scanned cards say ACCESS DENIED | The tag's actual UID does not match the hardcoded values | Scan the card using Project 84 first, read the hex output, and copy those exact bytes into `MASTER_UID` |
| Both LEDs turn on | Short circuit | Check breadboard wiring and verify distinct GPIO assignments |
| Sensor doesn't read | Reset pin issue | Verify RST is connected to GPIO 22 and initialised in code |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [84 - ESP32 MFRC522 RFID Card UID Serial Logs](84-esp32-mfrc522-rfid-card-uid-serial-logs.md)
- [86 - ESP32 MFRC522 RFID Keycard Access Buzzer](86-esp32-mfrc522-rfid-keycard-access-buzzer.md)
- [87 - ESP32 MFRC522 RFID Servo Door Unlock](87-esp32-mfrc522-rfid-servo-door-unlock.md)
