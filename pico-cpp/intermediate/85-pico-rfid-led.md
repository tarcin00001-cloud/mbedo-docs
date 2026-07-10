# 85 - Pico RFID LED

Build an RFID access checker that lights green for authorized cards and red for unauthorized cards.

## Goal
Learn how to match scanned RFID UIDs against hardcoded authorization values to trigger digital output indicators.

## What You Will Build
An RFID access control simulator:
- **MFRC522 RFID Reader**: Reads card UIDs.
- **Green LED (GP14)**: Turns ON for 2 seconds if the scanned card UID matches the authorized value (`DE 12 A4 C3`).
- **Red LED (GP15)**: Turns ON for 2 seconds if a card does not match the authorized value.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| Resistors | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MFRC522 | SDA (SS) | GP17 | Chip Select |
| MFRC522 | SCK | GP18 | SPI Clock |
| MFRC522 | MOSI | GP19 | Master Out Slave In |
| MFRC522 | MISO | GP16 | Master In Slave Out |
| MFRC522 | RST | GP22 | Reset line |
| Green LED | Anode | GP14 | Authorized status |
| Green LED | Cathode | GND | Ground return via resistor |
| Red LED | Anode | GP15 | Denied status |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>

const int RST_PIN = 22;
const int SS_PIN  = 17;
const int GRN_LED = 14;
const int RED_LED = 15;

MFRC522 mfrc522(SS_PIN, RST_PIN);

// Authorized card UID bytes: DE 12 A4 C3
const byte authUID[4] = {0xDE, 0x12, 0xA4, 0xC3};

void setup() {
  SPI.begin();
  mfrc522.PCD_Init();
  
  pinMode(GRN_LED, OUTPUT);
  pinMode(RED_LED, OUTPUT);

  // Initialize both LEDs OFF
  digitalWrite(GRN_LED, LOW);
  digitalWrite(RED_LED, LOW);
}

void loop() {
  if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
    
    // Check if scanned card matches authUID
    bool match = true;
    if (mfrc522.uid.uidByte[0] != authUID[0]) { match = false; }
    if (mfrc522.uid.uidByte[1] != authUID[1]) { match = false; }
    if (mfrc522.uid.uidByte[2] != authUID[2]) { match = false; }
    if (mfrc522.uid.uidByte[3] != authUID[3]) { match = false; }

    if (match) {
      // Access Granted
      digitalWrite(GRN_LED, HIGH);
      digitalWrite(RED_LED, LOW);
      delay(2000);
      digitalWrite(GRN_LED, LOW);
    } else {
      // Access Denied
      digitalWrite(GRN_LED, LOW);
      digitalWrite(RED_LED, HIGH);
      delay(2000);
      digitalWrite(RED_LED, LOW);
    }

    mfrc522.PICC_HaltA();
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MFRC522 RFID Reader**, **Green LED**, and **Red LED** onto the canvas.
2. Connect MFRC522 SPI pins: **SDA** to **GP17**, **SCK** to **GP18**, **MOSI** to **GP19**, **MISO** to **GP16**, **RST** to **GP22**.
3. Connect Green LED to **GP14**, Red LED to **GP15**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Test authorization by scanning matching and non-matching RFID tags.

## Expected Output

Terminal:
```
Simulation active. Access control engine running.
```

## Expected Canvas Behavior
* Scan Authorized Card (`DE 12 A4 C3`): Green LED (GP14) turns ON for 2 seconds.
* Scan Unauthorized Card: Red LED (GP15) turns ON for 2 seconds.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `mfrc522.uid.uidByte[0] != authUID[0]` | Compares the scanned UID bytes sequentially against the authorized list, avoiding forbidden interpreted loops. |

## Hardware & Safety Concept: Hardcoded Security Limits
While hardcoding authorized UID values is fine for simple educational projects, it is unsafe for real security systems. Scanned RFID cards are easy to clone if their UID is known. Real-world secure access readers read data from encrypted card sectors using custom keys rather than relying on public manufacturer UIDs.

## Try This! (Challenges)
1. **Denied Alarm**: Add a buzzer on GP10 and sound a warning tone only when access is denied.
2. **Access Log**: Log "ACCESS GRANTED" or "ACCESS DENIED" messages to the Serial Monitor.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Scanned card is always denied | UID byte mismatch | Print the card UID to Serial using project 84 to find the exact hex values, then paste them into the `authUID` array. |

## Mode Notes
This multi-device SPI project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [84 - Pico RFID Serial](84-pico-rfid-serial.md)
- [86 - Pico RFID Buzzer](86-pico-rfid-buzzer.md)
