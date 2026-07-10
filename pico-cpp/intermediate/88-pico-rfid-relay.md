# 88 - Pico RFID Relay

Control a high-power relay switch using validated RFID card key credentials.

## Goal
Learn how to actuate mechanical relay contacts to switch high-power loads using SPI-based RFID readers.

## What You Will Build
An industrial power switcher console:
- **MFRC522 RFID Reader**: Reads card UIDs.
- **Relay Module (GP10)**: Activates (NO closes) for 3 seconds when the authorized card (`DE 12 A4 C3`) is scanned, then deactivates.
- **Warning LED (GP15)**: Toggles ON to indicate the relay contact is active.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MFRC522 | SDA (SS) | GP17 | Chip Select |
| MFRC522 | SCK | GP18 | SPI Clock |
| MFRC522 | MOSI | GP19 | Master Out Slave In |
| MFRC522 | MISO | GP16 | Master In Slave Out |
| MFRC522 | RST | GP22 | Reset line |
| Relay Module | VCC | 5V | Coil power |
| Relay Module | IN | GP10 | Relay signal |
| Relay Module | GND | GND | Shared ground |
| Red LED | Anode | GP15 | Status indicator |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>

const int RST_PIN   = 22;
const int SS_PIN    = 17;
const int RELAY_PIN = 10;
const int RED_LED   = 15;

MFRC522 mfrc522(SS_PIN, RST_PIN);

const byte authUID[4] = {0xDE, 0x12, 0xA4, 0xC3};

void setup() {
  SPI.begin();
  mfrc522.PCD_Init();
  
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(RED_LED, OUTPUT);
  
  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(RED_LED, LOW);
}

void loop() {
  if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
    
    // Check if card matches authUID
    bool match = true;
    if (mfrc522.uid.uidByte[0] != authUID[0]) { match = false; }
    if (mfrc522.uid.uidByte[1] != authUID[1]) { match = false; }
    if (mfrc522.uid.uidByte[2] != authUID[2]) { match = false; }
    if (mfrc522.uid.uidByte[3] != authUID[3]) { match = false; }

    if (match) {
      // Access Granted - Toggle Relay ON
      digitalWrite(RELAY_PIN, HIGH);
      digitalWrite(RED_LED, HIGH);
      delay(3000); // Keep active for 3 seconds
      digitalWrite(RELAY_PIN, LOW);
      digitalWrite(RED_LED, LOW);
    } else {
      // Denied - flash warning LED
      digitalWrite(RED_LED, HIGH);
      delay(200);
      digitalWrite(RED_LED, LOW);
      delay(200);
      digitalWrite(RED_LED, HIGH);
      delay(200);
      digitalWrite(RED_LED, LOW);
    }

    mfrc522.PICC_HaltA();
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MFRC522 RFID Reader**, **Relay Module**, and **Red LED** onto the canvas.
2. Connect MFRC522 SPI pins: **SDA** to **GP17**, **SCK** to **GP18**, **MOSI** to **GP19**, **MISO** to **GP16**, **RST** to **GP22**.
3. Connect Relay to **GP10**, LED to **GP15**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Scan authorized/unauthorized cards to switch the relay contacts.

## Expected Output

Terminal:
```
Simulation active. Industrial RFID switcher online.
```

## Expected Canvas Behavior
* Scan Authorized Card: Relay closes (GP10 HIGH) and LED turns ON for 3 seconds.
* Scan Unauthorized Card: LED flashes twice rapidly, relay remains open.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(RELAY_PIN, HIGH)` | Drives GP10 HIGH to energize the relay's electromagnetic coil, switching the internal contact pole. |

## Hardware & Safety Concept: AC Switching Safety
Relays let you control high-voltage AC mains appliances (like 220V lights, heaters, or motors) from a safe 3.3V microcontroller circuit. However, working with AC mains voltage poses severe shock and fire hazards. Real-world high-voltage wiring must be done in enclosed, insulated plastic junction boxes, ensuring zero exposed copper joints can touch fingers or metal frames.

## Try This! (Challenges)
1. **Latching Power Switch**: Modify the code so that scanning the authorized card once turns the relay ON permanently, and scanning it again turns it OFF (toggles).
2. **Audio verification**: Sound a buzzer on GP14 briefly during authorized scans.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay does not switch on authorized scan | Insufficient power supply | Ensure the relay VCC pin is wired to the Pico's 5V pin (VBUS), as 3.3V may not be enough to actuate the relay coil. |

## Mode Notes
This multi-device SPI project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [11 - Pico Relay Toggle](../../beginner/11-pico-relay-toggle.md)
- [85 - Pico RFID LED](85-pico-rfid-led.md)
