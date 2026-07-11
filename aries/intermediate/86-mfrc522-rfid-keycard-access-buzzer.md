# 86 - MFRC522 RFID Keycard Access Buzzer

Build a security access gate indicator that verifies RFID cards, playing a happy chime for authorized keys and sounding an error alarm for denied attempts.

## Goal
Learn how to store authorization codes, compare scanned tag bytes against master keys, and control an active buzzer to play warning tunes without using loops inside C++ code.

## What You Will Build
An MFRC522 reader connected via SPI. The code holds a target 4-byte master key UID. If the scanned card matches the key, a happy chime sounds on an Active Buzzer connected to GPIO 14. Otherwise, a long error beep is triggered.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MFRC522 RFID Reader Module | `rfid_reader` | Yes | Yes |
| RFID Key Fob / Card (13.56MHz) | `rfid_tag` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes (on GPIO 14) |

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
| Active Buzzer | Positive (+) | GPIO 14 | Red | Buzzer control line |
| Active Buzzer | Negative (-) | GND | Black | Ground reference |

> **Wiring tip:** Active buzzers are polarized devices. Ensure the longer pin or positive (+) marked terminal connects to GPIO 14 and the other terminal connects to GND.

## Code
```cpp
// MFRC522 RFID Keycard Access Buzzer
#include <SPI.h>
#include <MFRC522.h>

#define SS_PIN 10
#define RST_PIN 9

const int BUZZER_PIN = 14;

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
  
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW); // Start with buzzer OFF
  
  Serial.println("Chime Access system active. Scan Card...");
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
    Serial.println("Authorized! Playing Happy Chime.");
    
    // Play happy chime: 2 short beeps followed by a medium beep
    digitalWrite(BUZZER_PIN, HIGH);
    delay(80);
    digitalWrite(BUZZER_PIN, LOW);
    delay(50);
    digitalWrite(BUZZER_PIN, HIGH);
    delay(80);
    digitalWrite(BUZZER_PIN, LOW);
    delay(50);
    digitalWrite(BUZZER_PIN, HIGH);
    delay(200);
    digitalWrite(BUZZER_PIN, LOW);
  } else {
    Serial.println("Unauthorized! Playing Error Alarm.");
    
    // Play error beep: 1 long low beep
    digitalWrite(BUZZER_PIN, HIGH);
    delay(600);
    digitalWrite(BUZZER_PIN, LOW);
  }
  
  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **MFRC522 RFID Reader**, and **Active Buzzer** onto the canvas.
2. Wire the MFRC522 SPI lines as listed. Connect the Buzzer positive to **GPIO 14**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.
6. Drag a card tag to the reader widget. If its UID is `83 A2 C3 E4` the Buzzer plays the happy chime. If it differs, the Buzzer plays the error alarm.

## Expected Output
Serial Monitor:
```
Chime Access system active. Scan Card...
Scanned UID: 83 A2 C3 E4
Authorized! Playing Happy Chime.
Scanned UID: 12 34 56 78
Unauthorized! Playing Error Alarm.
```

## Expected Canvas Behavior
* Scanning the matching card tag makes the active buzzer widget play a distinct quick chime.
* Scanning any non-matching card tag triggers a solid long warning beep.
* The buzzer remains off otherwise.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(BUZZER_PIN, OUTPUT)` | Configures GPIO 14 as buzzer control output. |
| `rfid.uid.uidByte[0] == MASTER_0` | Directly evaluates if the first scanned byte matches the master key byte. |
| `digitalWrite(BUZZER_PIN, HIGH)` | Powers the buzzer coil to play tone segments. |
| `delay(80)` | Controls chime note durations. |

## Hardware & Safety Concept
* **Audio Telemetry Feedback**: Using auditory feedback helps users understand system states instantly. The buzzer plays different rhythms: high-tempo signals represent success or acceptance, while low-tempo continuous sounds represent failure or lockout states. This is standard in access gates, POS systems, and terminal locks.

## Try This! (Challenges)
1. **Visual and Audio Sync**: Connect a Warning LED to GPIO 15. Flash the LED in sync with the buzzer chime notes.
2. **Door Lock Delay**: Integrate a simulated locking time. Keep a Warning LED ON for 5 seconds after the happy chime plays to show the gate is unlocked.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer does not sound | Polarity reversed or wrong pin | Verify the positive buzzer lead connects to GPIO 14. |
| All scanned cards play Error Alarm | Scanned card doesn't match MASTER | Check actual card UID using Project 84, and update code values. |
| Reader does not read cards | SPI wiring loose | Double-check that SCK, MISO, MOSI, and SS connections are secure. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [84 - MFRC522 RFID Card UID Serial Logs](84-mfrc522-rfid-card-uid-serial.md)
- [85 - MFRC522 RFID Keycard Access LED](85-mfrc522-rfid-keycard-access-led.md)
- [87 - MFRC522 RFID Servo Door Unlock](87-mfrc522-rfid-servo-door-unlock.md)
