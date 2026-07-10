# 86 - Pico RFID Buzzer

Build an RFID access checker that plays success/error chime tones on card scan.

## Goal
Learn how to use scanned RFID UIDs to trigger different acoustic chimes based on authorization matches.

## What You Will Build
An acoustic access indicator:
- **MFRC522 RFID Reader**: Reads card UIDs.
- **Passive Buzzer (GP14)**: Plays a happy two-tone success chime (660Hz → 880Hz) on authorized scans (`DE 12 A4 C3`), and a low error buzz (150Hz) on denied scans.
- **Green LED (GP10)**: Lights during success chimes.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MFRC522 | SDA (SS) | GP17 | Chip Select |
| MFRC522 | SCK | GP18 | SPI Clock |
| MFRC522 | MOSI | GP19 | Master Out Slave In |
| MFRC522 | MISO | GP16 | Master In Slave Out |
| MFRC522 | RST | GP22 | Reset line |
| Passive Buzzer | VCC (+) | GP14 | Sounder control |
| Passive Buzzer | GND (-) | GND | Ground return |
| Green LED | Anode | GP10 | Status indicator |
| Green LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>

const int RST_PIN    = 22;
const int SS_PIN     = 17;
const int BUZZER_PIN = 14;
const int GRN_LED    = 10;

MFRC522 mfrc522(SS_PIN, RST_PIN);

const byte authUID[4] = {0xDE, 0x12, 0xA4, 0xC3};

void setup() {
  SPI.begin();
  mfrc522.PCD_Init();
  
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(GRN_LED, OUTPUT);
  digitalWrite(GRN_LED, LOW);
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
      // Access Granted Success Chime
      digitalWrite(GRN_LED, HIGH);
      tone(BUZZER_PIN, 660, 150);
      delay(150);
      tone(BUZZER_PIN, 880, 300);
      delay(300);
      digitalWrite(GRN_LED, LOW);
    } else {
      // Access Denied Error Tone
      digitalWrite(GRN_LED, LOW);
      tone(BUZZER_PIN, 150, 600);
      delay(600);
    }

    mfrc522.PICC_HaltA();
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MFRC522 RFID Reader**, **Passive Buzzer**, and **Green LED** onto the canvas.
2. Connect MFRC522 SPI pins: **SDA** to **GP17**, **SCK** to **GP18**, **MOSI** to **GP19**, **MISO** to **GP16**, **RST** to **GP22**.
3. Connect Buzzer to **GP14**, LED to **GP10**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Scan authorized and unauthorized tags to hear the audio feedback chimes.

## Expected Output

Terminal:
```
Simulation active. Audio authorization engine online.
```

## Expected Canvas Behavior
* Scan Authorized Card: LED (GP10) turns ON, buzzer plays success chime (high tones).
* Scan Unauthorized Card: LED stays OFF, buzzer plays error tone (low buzz).

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `tone(BUZZER_PIN, 660, 150)` | Plays a 660 Hz tone (E5 note) for 150 ms to output acoustic success cues. |

## Hardware & Safety Concept: User Interface Sound Cues
In user interface design, visual feedback (like LED colors) should always be paired with auditory feedback (sound chimes). This ensures the device is accessible to users with visual impairments. Using distinct tone patterns (high-pitch double beeps for success, low-pitch buzzes for errors) helps users understand system states immediately without looking at screens.

## Try This! (Challenges)
1. **Siren Delay**: Add a Red LED on GP15 and configure it to flash on unauthorized scans.
2. **Doorbell Tone**: Program a custom three-note chime on authorized access.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sound output is high-pitch click only | Active buzzer used | Confirm you are using a **passive** buzzer. Active buzzers cannot produce high/low pitch changes on frequency requests. |

## Mode Notes
This multi-device SPI project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [70 - Pico Buzzer Scale](70-pico-buzzer-scale.md)
- [85 - Pico RFID LED](85-pico-rfid-led.md)
