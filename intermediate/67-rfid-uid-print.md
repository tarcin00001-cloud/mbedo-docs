# 67 - RFID UID Print

Read and display the unique identification number (UID) of RFID cards and keyfobs.

## Goal
Learn how to initialize the SPI (Serial Peripheral Interface) bus and control an MFRC522 RFID reader module to extract card serial numbers.

## What You Will Build
When an RFID card or keyfob is swiped near the RFID reader, the Arduino reads its unique identifier bytes and prints them as a formatted hexadecimal sequence (e.g. `Card UID: DE AD BE EF`) to the Terminal.

**Why SPI Pins and D9, D10?** Pins D11, D12, and D13 comprise the standard hardware SPI lines (MOSI, MISO, SCK) on the Uno. Pin D10 acts as the SS (Slave Select) line and D9 controls the reset line.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| MFRC522 Module | SDA (SS) | D10 | SPI Slave Select |
| MFRC522 Module | SCK | D13 | SPI Serial Clock |
| MFRC522 Module | MOSI | D11 | SPI Master Out Slave In |
| MFRC522 Module | MISO | D12 | SPI Master In Slave Out |
| MFRC522 Module | RST | D9 | Reset control line |
| MFRC522 Module | VCC | 5V | Power supply (See Safety Note below) |
| MFRC522 Module | GND | GND | Ground reference |

> [!CAUTION]
> **Hardware Voltage Warning:** Real-world MFRC522 breakout modules operate strictly on **3.3V**. Connecting a physical module directly to 5V will permanently burn out the chip. Always connect VCC to the Arduino's 3.3V pin on physical hardware. (The MbedO canvas catalog allows 5V wiring for simulation convenience).

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>

#define SS_PIN 10
#define RST_PIN 9

MFRC522 rfid(SS_PIN, RST_PIN);

void setup() {
  Serial.begin(9600);
  
  // Start the SPI bus
  SPI.begin();
  
  // Start the RFID reader module
  rfid.PCD_Init();
  
  Serial.println("RFID Reader Ready. Scan a card/tag...");
}

void loop() {
  // Check if a new card is present near the antenna
  if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
    Serial.print("Card UID:");
    
    // Print the UID bytes in hex format
    for (byte i = 0; i < rfid.uid.size; i++) {
      Serial.print(rfid.uid.uidByte[i] < 0x10 ? " 0" : " ");
      Serial.print(rfid.uid.uidByte[i], HEX);
    }
    Serial.println();
    
    // Stop reading this card to prevent duplicate loops
    rfid.PICC_HaltA();
    rfid.PCD_StopCrypto1();
  }
  
  delay(100);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** and **RFID Reader (MFRC522)** onto the canvas.
2. Wire the module: **SDA** to **D10**, **SCK** to **D13**, **MOSI** to **D11**, **MISO** to **D12**, **RST** to **D9**, **VCC** to **5V**, and **GND** to **GND**.
3. Paste the code into the editor.
4. Use the default Arduino interpreted runtime.
5. Click **Run**.
6. Open the **Terminal** tab in the bottom dock.
7. Click the RFID Reader component on the canvas to trigger a simulated card swipe.
8. Watch the card UID output print in the Terminal.

## Expected Output

Terminal:
```
RFID Reader Ready. Scan a card/tag...
Card UID: DE AD BE EF
...
```

### Expected Canvas Behavior

| Canvas Action | Pin Activity | Terminal Output |
| --- | --- | --- |
| Idle | No signals | None |
| Click Reader (Swipe) | SPI communication active | "Card UID: DE AD BE EF" |

Swiping a card displays its serial bytes in hexadecimal format in the Terminal.

## Code Walkthrough

| Class / Function | What It Does |
| --- | --- |
| `SPI.begin()` | Sets up the Serial Peripheral Interface bus registers. |
| `rfid.PCD_Init()` | Resets and configures the reader module antenna. |
| `rfid.PICC_IsNewCardPresent()` | Polls the antenna field to detect if an radio tag is nearby. |
| `rfid.PICC_ReadCardSerial()` | Instructs the card to transmit its internal UID array. Returns true if successful. |
| `rfid.uid.uidByte[i]` | Contains the raw bytes of the card serial number. |

## Hardware & Safety Concept: How RFID Works
Radio Frequency Identification (RFID) uses electromagnetic fields to transmit data.
- The reader's antenna loop generates a high-frequency **13.56 MHz** electromagnetic field.
- The RFID card contains a coil antenna and an integrated microchip. It has no battery (passive tag).
- When placed in the reader's field, the card's coil absorbs enough energy via **induction** to power the microchip and transmit its 64-bit registration UID back over the air.

## Try This! (Challenges)
1. **Decimal Output**: Modify the loop to print the UID bytes in decimal format instead of hexadecimal.
2. **Access Granted Banner**: If the UID matches `DE AD BE EF`, print "Access Granted", otherwise print "Access Denied".

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Card scans but prints empty values | SPI wires crossed | Verify that D11 connects to MOSI and D12 connects to MISO. If reversed, the Arduino cannot receive bytes. |
| Tag won't scan at all | Antenna range limits | On physical hardware, passive cards must be held within 2-4 cm of the reader loop antenna. |

## Mode Notes
The `MFRC522` and `SPI` libraries are fully supported by shims in the MbedO interpreted mode runtime.

## Related Projects
- [68 - RFID Access LED](68-rfid-access-led.md)
- [69 - RFID Access Buzzer](69-rfid-access-buzzer.md)
