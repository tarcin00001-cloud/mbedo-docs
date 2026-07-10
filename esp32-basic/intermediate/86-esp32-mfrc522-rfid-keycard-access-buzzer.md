# 86 - ESP32 MFRC522 RFID Keycard Access Buzzer

Build an RFID access checker that outputs audio feedback: a high-pitched double chirp for success and a low-pitched long buzz for failure.

## Goal
Learn how to use audio indicators to convey access status notifications without requiring a display screen, using the `tone()` utility function.

## What You Will Build
An MFRC522 RFID reader connected via SPI. A passive buzzer on GPIO 15 generates distinct tones based on authorisation: two quick 2000 Hz chirps for access granted, and one long 300 Hz tone for access denied.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MFRC522 RFID Reader Module | `rfid_reader` | Yes | Yes |
| RFID Key Fob / Card (13.56MHz) | `rfid_tag` | Yes | Yes |
| Passive Buzzer (Piezo Speaker) | `buzzer` | Yes | Yes |

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
| Passive Buzzer | Positive (+) | GPIO15 | Blue | Audio signal line |
| Passive Buzzer | Negative (−) | GND | Black | Ground reference |

> **Wiring tip:** The passive buzzer is wired to GPIO 15. The MFRC522 pins are identical to the previous RFID projects.

## Code
```cpp
// MFRC522 RFID Keycard Access Buzzer
#include <SPI.h>
#include <MFRC522.h>

#define SS_PIN 5
#define RST_PIN 22

const int BUZZER_PIN = 15;

MFRC522 rfid(SS_PIN, RST_PIN);

// Authorised Master Key UID
const byte MASTER_UID[] = {0x83, 0xA2, 0xC3, 0xE4};

void setup() {
  Serial.begin(115200);
  SPI.begin();
  rfid.PCD_Init();
  
  pinMode(BUZZER_PIN, OUTPUT);
  noTone(BUZZER_PIN);
  
  Serial.println("RFID Sound Access station active.");
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
    Serial.println("Access Granted!");
    // Success Chime: 2 high beeps
    tone(BUZZER_PIN, 2000);
    delay(100);
    noTone(BUZZER_PIN);
    delay(50);
    tone(BUZZER_PIN, 2000);
    delay(100);
    noTone(BUZZER_PIN);
  } else {
    Serial.println("Access Denied!");
    // Failure Chime: 1 long low tone
    tone(BUZZER_PIN, 300);
    delay(600);
    noTone(BUZZER_PIN);
  }
  
  delay(1000); // 1 second cool-down delay
  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MFRC522 Reader**, and **Buzzer** onto the canvas.
2. Wire the SPI reader pins. Connect the buzzer positive pin to **GPIO15**.
3. Paste the code and click **Run**.
4. Drag a card tag to the reader widget. Listen to the different chime pitches.

## Expected Output
Serial Monitor:
```
RFID Sound Access station active.
Access Granted!
Access Denied!
```

## Expected Canvas Behavior
* Scanning the matching card tag plays a rapid double chirp (2000 Hz) from the speaker.
* Scanning any other card tag plays a flat, low-pitch buzz (300 Hz) for 600 ms.
* The system rests silent between card reads.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `tone(BUZZER_PIN, 2000)` | Generates a 2 kHz high frequency chime on GPIO 15. |
| `tone(BUZZER_PIN, 300)` | Generates a 300 Hz low frequency warning buzz on GPIO 15. |
| `noTone(BUZZER_PIN)` | Stops the generated square wave immediately, silencing the buzzer. |

## Hardware & Safety Concept: User Ergonomics and Sound Indicators
Using distinct audio tones (pitches and durations) allows visually impaired users to interact with secure systems. A high-pitched chirp represents success, while a low-pitched flat tone universally denotes errors or warning alerts.

## Try This! (Challenges)
1. **Combine LED & Sound**: Connect Red/Green LEDs (Project 85) alongside the buzzer to create a full audio-visual entry system.
2. **Keypad chime scale**: Play an ascending arpeggio on success and a descending scale on failure.
3. **Lockout warning**: If a card is denied 3 times in a row, play a continuous alarm sound for 5 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sound is clicking or weak | Active buzzer used instead of passive | Swap buzzer type, active buzzers cannot change frequency |
| Sound stays on continuously | Missing `noTone()` calls | Make sure `noTone()` is called after each `delay()` window |
| Scanning does not respond | SPI bus conflicts | Verify SPI chip select pin connections are secure |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [85 - ESP32 MFRC522 RFID Keycard Access LED](85-esp32-mfrc522-rfid-keycard-access-led.md)
- [87 - ESP32 MFRC522 RFID Servo Door Unlock](87-esp32-mfrc522-rfid-servo-door-unlock.md)
- [70 - ESP32 Active Buzzer Chime Scale Generator](70-esp32-active-buzzer-chime-scale-generator.md)
