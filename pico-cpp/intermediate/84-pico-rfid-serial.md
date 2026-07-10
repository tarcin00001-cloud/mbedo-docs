# 84 - Pico RFID Serial

Read and log passive RFID card UID address bytes to the Serial Monitor using an MFRC522 module.

## Goal
Learn how to interface SPI devices, read card registry blocks, and translate hex byte arrays to the Serial Monitor.

## What You Will Build
An RFID card detector:
- **MFRC522 RFID Reader (SPI GP16-GP19)**: Reads card/key fob UIDs.
- **Serial Monitor**: Prints the card UID in hexadecimal format (e.g. "Card UID: A3 B2 4C D5") each time a card approaches the reader.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| Mifare 1K Card / Fob | `rfid_tag` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MFRC522 | 3.3V | 3.3V | Power supply |
| MFRC522 | RST (Reset) | GP22 | Reset line |
| MFRC522 | GND | GND | Ground reference |
| MFRC522 | MISO | GP16 (SPI0 RX) | Master In Slave Out |
| MFRC522 | MOSI | GP19 (SPI0 TX) | Master Out Slave In |
| MFRC522 | SCK | GP18 (SPI0 SCK)| Serial Clock |
| MFRC522 | SDA (SS) | GP17 (SPI0 CS) | Chip Select |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>

const int RST_PIN = 22;
const int SS_PIN  = 17;

MFRC522 mfrc522(SS_PIN, RST_PIN);

void setup() {
  Serial.begin(9600);
  SPI.begin();
  mfrc522.PCD_Init();
  Serial.println("RFID Reader Online - Scan Card");
}

void loop() {
  // Check if a new card is present
  if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
    Serial.print("Card UID: ");
    
    // Print UID bytes in hex format (avoiding loops by hardcoding index)
    Serial.print(mfrc522.uid.uidByte[0], HEX); Serial.print(" ");
    Serial.print(mfrc522.uid.uidByte[1], HEX); Serial.print(" ");
    Serial.print(mfrc522.uid.uidByte[2], HEX); Serial.print(" ");
    Serial.print(mfrc522.uid.uidByte[3], HEX);
    Serial.println();

    // Halt PICC to stop repeated readings
    mfrc522.PICC_HaltA();
  }
  
  delay(100);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **MFRC522 RFID Reader** onto the canvas.
2. Connect MFRC522: **3.3V** to **3V3**, **RST** to **GP22**, **GND** to **GND**, **MISO** to **GP16**, **MOSI** to **GP19**, **SCK** to **GP18**, **SDA** to **GP17**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Drag an RFID card or key fob near the reader model and view the Serial logs.

## Expected Output

Terminal:
```
RFID Reader Online - Scan Card
Card UID: DE 12 A4 C3
Card UID: F5 88 BB AA
```

## Expected Canvas Behavior
* Approaching Card 1 prints: `Card UID: DE 12 A4 C3`
* Approaching Card 2 prints: `Card UID: F5 88 BB AA`

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `mfrc522.PCD_Init()` | Initializes the internal registers of the MFRC522 reader chip to prepare the antenna for RF transmission. |
| `mfrc522.uid.uidByte[0]` | Accesses the first byte of the card's Unique Identifier array stored in the MFRC522 library buffer. |
| `mfrc522.PICC_HaltA()` | Puts the active card into a sleep state, preventing it from constantly responding while in range of the antenna. |

## Hardware & Safety Concept: SPI Chip Select (CS)
SPI (Serial Peripheral Interface) is a synchronous, full-duplex serial protocol. The master (Pico) uses three shared lines (SCK, MOSI, MISO) to communicate with multiple slave devices. To select a specific device, the master pulls that device's dedicated **SS (Slave Select / Chip Select)** line `LOW`. When the transaction finishes, the SS line is pulled `HIGH`, letting other devices use the shared SPI bus.

## Try This! (Challenges)
1. **Access Authorized LED**: Add a Red LED on GP15 and turn it ON for 1 second when *any* card is scanned.
2. **Alert Tone**: Sound a buzzer on GP14 briefly when a card is detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| RFID reader fails to read cards | Swapped SPI lines | Verify MISO (GP16) and MOSI (GP19) connections. Swapping data pins is a common error in SPI setups. |

## Mode Notes
This basic SPI communication project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [85 - Pico RFID LED](85-pico-rfid-led.md)
- [87 - Pico RFID Servo](87-pico-rfid-servo.md)
