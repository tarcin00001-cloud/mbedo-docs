# 68 - RFID Access LED

Build an RFID access gate indicator that lights a Green LED for approved cards and a Red LED for unapproved cards.

## Goal
Learn how to compare scanned array bytes against a stored Master key card signature to trigger digital outputs.

## What You Will Build
When a card is swiped:
- If the card UID matches the master key (`DE AD BE EF`), the Green LED on D8 turns ON for 2 seconds (representing access granted).
- If it does not match, the Red LED on D7 flashes three times (representing access denied).

**Why D7 and D8?** Pins D7 and D8 control the red and green warning LEDs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| Green LED | `led` | Yes | Yes |
| Red LED | `led` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes (2 required) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| MFRC522 Module | SDA (SS) | D10 | SPI Slave Select |
| MFRC522 Module | SCK | D13 | SPI Clock |
| MFRC522 Module | MOSI | D11 | SPI MOSI |
| MFRC522 Module | MISO | D12 | SPI MISO |
| MFRC522 Module | RST | D9 | Reset |
| MFRC522 Module | VCC | 5V | Power supply (Caution: 3.3V on hardware) |
| MFRC522 Module | GND | GND | Ground |
| Green LED | A | D8 | Access approved indicator |
| Green LED | C | GND | Ground |
| Red LED | A | D7 | Access denied indicator |
| Red LED | C | GND | Ground |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>

#define SS_PIN 10
#define RST_PIN 9
#define RED_LED 7
#define GREEN_LED 8

MFRC522 rfid(SS_PIN, RST_PIN);

// Authorized Master Card UID (adjust to match your card's UID)
const byte MASTER_UID[] = {0xDE, 0xAD, 0xBE, 0xEF};

void setup() {
  pinMode(RED_LED, OUTPUT);
  pinMode(GREEN_LED, OUTPUT);
  
  Serial.begin(9600);
  SPI.begin();
  rfid.PCD_Init();
  
  Serial.println("Access System Ready. Scan card...");
}

void loop() {
  // Check for new cards
  if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
    Serial.print("Scanned card UID:");
    for (byte i = 0; i < rfid.uid.size; i++) {
      Serial.print(" ");
      Serial.print(rfid.uid.uidByte[i], HEX);
    }
    Serial.println();
    
    // Compare scanned card UID against our authorized Master UID
    bool isAuthorized = true;
    for (byte i = 0; i < 4; i++) {
      if (rfid.uid.uidByte[i] != MASTER_UID[i]) {
        isAuthorized = false;
        break;
      }
    }
    
    if (isAuthorized) {
      Serial.println("[SUCCESS] Access Granted! Door unlocked.");
      digitalWrite(GREEN_LED, HIGH);
      delay(2000); // Keep door open for 2 seconds
      digitalWrite(GREEN_LED, LOW);
    } else {
      Serial.println("[DENIED] Access Blocked!");
      
      // Flash Red LED 3 times
      for (int k = 0; k < 3; k++) {
        digitalWrite(RED_LED, HIGH);
        delay(150);
        digitalWrite(RED_LED, LOW);
        delay(150);
      }
    }
    
    rfid.PICC_HaltA();
    rfid.PCD_StopCrypto1();
  }
  
  delay(100);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **RFID Reader (MFRC522)**, and **two LEDs** (one red, one green) onto the canvas.
2. Connect MFRC522: **SDA** to **D10**, **SCK** to **D13**, **MOSI** to **D11**, **MISO** to **D12**, **RST** to **D9**, **VCC** to **5V**, and **GND** to **GND**.
3. Connect Green LED **A** to Arduino **D8**, Red LED **A** to Arduino **D7**, and both **C** pins to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Click the RFID Reader component to trigger a simulated card swipe.
8. Watch the Green LED light up, indicating a match on the default `DE AD BE EF` UID.

## Expected Output

Terminal:
```
Access System Ready. Scan card...
Scanned card UID: DE AD BE EF
[SUCCESS] Access Granted! Door unlocked.
...
```

### Expected Canvas Behavior

| Swiped Card UID | Match Check | Pin state action | Visual indicator |
| --- | --- | --- | --- |
| DE AD BE EF | Match | D8 HIGH for 2s | Green LED lights up |
| Other (e.g. 12 34 56 78) | Mismatch | D7 pulses 3 times | Red LED flashes |

The Green LED lights up steady on the canvas when the default card is swiped.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `const byte MASTER_UID[] = {0xDE, 0xAD, 0xBE, 0xEF};` | Stores the authorized UID keys array to match. |
| `rfid.uid.uidByte[i] != MASTER_UID[i]` | Compares scanned bytes against the stored values, setting `isAuthorized` to false if a mismatch is found. |

## Hardware & Safety Concept: Secure Memory Comparison
Access control nodes must store their master keys securely.
- In simple projects, master keys are hardcoded in the Arduino's flash memory.
- In production card systems, keys are stored in the Arduino's **EEPROM** (Electrically Erasable Programmable Read-Only Memory) so they are preserved when the board loses power, allowing admins to add or remove cards without rewriting the code.

## Try This! (Challenges)
1. **Change Master Key**: Adjust the `MASTER_UID` array in the code to require a different card ID.
2. **Reverse lock**: Make the Red LED remain ON normally (representing a locked state) and turn OFF only when the Green LED lights up.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Green LED does not light up on swipe | UID mismatch | Verify the scanned card UID printed in the Terminal matches the `MASTER_UID` array defined in the code. |
| LEDs never light up | Resistors or LED polarities reversed | Ensure anode (+) connects to D7/D8 and cathode (-) to GND. |

## Mode Notes
These patterns (array loops, comparison conditionals, and digital writes) are supported by MbedO interpreted mode.

## Related Projects
- [67 - RFID UID Print](67-rfid-uid-print.md)
- [69 - RFID Access Buzzer](69-rfid-access-buzzer.md)
- [70 - RFID Servo Unlock](70-rfid-servo-unlock.md)
