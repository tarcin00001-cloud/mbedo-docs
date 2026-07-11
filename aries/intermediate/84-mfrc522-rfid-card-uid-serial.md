# 84 - MFRC522 RFID Card UID Serial Logs

Scan RFID tags using an MFRC522 reader module and print their unique UID numbers in hexadecimal format to the Serial Monitor.

## Goal
Learn how to interface SPI bus communication devices, initialize the MFRC522 card reader, scan high-frequency RFID transponders, and parse binary UIDs without using loops inside C++ code.

## What You Will Build
An MFRC522 RFID reader is connected to the ARIES v3 board using the SPI bus interface. When an RFID card or key fob is brought near the reader, the board reads the card's Unique Identifier (UID), prints the card type, and outputs the hex code to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MFRC522 RFID Reader Module | `rfid_reader` | Yes | Yes |
| RFID Key Fob / Card (13.56MHz) | `rfid_tag` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 Module | 3.3V | 3V3 | Red | Power (DO NOT connect to 5V) |
| MFRC522 Module | GND | GND | Black | Ground reference |
| MFRC522 Module | RST (Reset) | GPIO 9 | Purple | Reset line |
| MFRC522 Module | SDA (SS) | GPIO 10 | Orange | Chip Select |
| MFRC522 Module | SCK | SPI SCK | Yellow | SPI Clock line |
| MFRC522 Module | MISO | SPI MISO | Green | SPI Master In Slave Out |
| MFRC522 Module | MOSI | SPI MOSI | Blue | SPI Master Out Slave In |

> **Wiring tip:** The MFRC522 module operates strictly on **3.3V**. Connecting VCC to 5V will destroy the reader chip. Ensure all SPI connections are correct.

## Code
```cpp
// MFRC522 RFID Card UID Serial Logs
#include <SPI.h>
#include <MFRC522.h>

#define SS_PIN 10
#define RST_PIN 9

MFRC522 rfid(SS_PIN, RST_PIN);

void setup() {
  Serial.begin(115200);
  SPI.begin();       // Initialize SPI bus
  rfid.PCD_Init();   // Initialize MFRC522 reader
  
  Serial.println("MFRC522 RFID Scanner Active. Scan a card/tag...");
}

void loop() {
  // Look for new cards
  if ( ! rfid.PICC_IsNewCardPresent()) {
    return;
  }

  // Verify if the UID has been read
  if ( ! rfid.PICC_ReadCardSerial()) {
    return;
  }

  // Print card type
  MFRC522::PICC_Type piccType = rfid.PICC_GetType(rfid.uid.sak);
  Serial.print("RFID Tag Detected! Type: ");
  Serial.println(rfid.PICC_GetTypeName(piccType));

  // Print 4-byte UID in Hexadecimal format without using loops
  Serial.print("Card UID: ");
  Serial.print(rfid.uid.uidByte[0], HEX);
  Serial.print(" ");
  Serial.print(rfid.uid.uidByte[1], HEX);
  Serial.print(" ");
  Serial.print(rfid.uid.uidByte[2], HEX);
  Serial.print(" ");
  Serial.println(rfid.uid.uidByte[3], HEX);
  Serial.println("----------------------------------------");

  // Halt PICC to stop repeated readings
  rfid.PICC_HaltA();
  // Stop encryption on PCD
  rfid.PCD_StopCrypto1();
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** and **MFRC522 RFID Reader** components onto the canvas.
2. Wire the MFRC522 SPI lines as specified in the wiring table. Connect **RST** to **GPIO 9** and **SDA (SS)** to **GPIO 10**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.
6. Drag an RFID Tag component on the canvas near the reader widget. Watch the Serial Monitor logs.

## Expected Output
Serial Monitor:
```
MFRC522 RFID Scanner Active. Scan a card/tag...
RFID Tag Detected! Type: MIFARE Classic 1K
Card UID: 83 A2 C3 E4
----------------------------------------
```

## Expected Canvas Behavior
* Bringing a virtual RFID tag card widget close to the reader widget triggers the detection logic.
* The Serial Monitor prints the card type and UID value instantly.
* The system waits until the card is removed and scanned again to print next logs.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `SPI.begin()` | Initialises the Serial Peripheral Interface bus. |
| `rfid.PCD_Init()` | Powers up and configures the MFRC522 chip registers. |
| `rfid.PICC_IsNewCardPresent()` | Non-blocking poll checking if an RF card enters the antenna's magnetic field. |
| `rfid.PICC_ReadCardSerial()` | Reads the card serial number registry (UID). Returns true if successful. |
| `rfid.uid.uidByte[0]` | Accesses the first byte of the card's array UID. |
| `rfid.PICC_HaltA()` | Puts the tag in sleep state so it doesn't trigger continuous readings while resting on the antenna. |

## Hardware & Safety Concept
* **RFID Near-Field Communication**: The MFRC522 reader uses a 13.56 MHz electromagnetic field to power passive tags. When a card gets close, the card's internal coil antenna harvests this energy through electromagnetic induction to power its onboard microchip. The chip then modulates the magnetic field to send its stored UID code back to the reader. RFID is highly secure, contactless, and unaffected by dirt, grease, or visual occlusion.

## Try This! (Challenges)
1. **Access Authorized check**: Save your card's UID bytes using individual variables. Print "ACCESS GRANTED" if that specific card is scanned, and "ACCESS DENIED" for others.
2. **Beep on Scan**: Connect a buzzer to GPIO 14 and beep briefly (100 ms) every time a card is successfully scanned.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| RFID reader fails to start | SPI pins wired incorrectly | Verify MISO, MOSI, SCK, and CS (GPIO 10) pins are correctly connected. |
| Reading fails or drops out | Power supply issue | Verify 3V3 power is stable. Do not share 3.3V line with high-draw motors. |
| Card detected but UID is empty | Scan range is too far | Keep tags within 0 to 4 cm of the reader antenna. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [85 - MFRC522 RFID Keycard Access LED](85-mfrc522-rfid-keycard-access-led.md)
- [87 - MFRC522 RFID Servo Door Unlock](87-mfrc522-rfid-servo-door-unlock.md)
- [88 - MFRC522 RFID Relay Control](88-mfrc522-rfid-relay-control.md)
