# 71 - RFID Relay Door

Control a high-voltage electromagnetic door strike using an RFID reader and a Relay module.

## Goal
Learn how to interface an RFID card reader with an isolation relay module to safely switch external high-power door strike locks.

## What You Will Build
When a card is swiped:
- If the card UID matches the master key (`DE AD BE EF`), the relay connected to D4 triggers ON (closed contacts) for 3 seconds, simulating unlocking a commercial electromagnetic lock.
- If it does not match, the relay remains OFF.

**Why D4?** Pin D4 triggers the transistor switch inside the relay module.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| 1N4007 Diode | `diode` | Optional | Yes (critical flyback diode) |

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
| Relay Module | IN | D4 | Control pin |
| Relay Module | VCC | 5V | Relay coil power |
| Relay Module | GND | GND | Ground reference |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>

#define SS_PIN 10
#define RST_PIN 9
#define RELAY_PIN 4

MFRC522 rfid(SS_PIN, RST_PIN);

// Authorized Master Card UID
const byte MASTER_UID[] = {0xDE, 0xAD, 0xBE, 0xEF};

void setup() {
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Keep relay off (locked) at boot
  
  Serial.begin(9600);
  SPI.begin();
  rfid.PCD_Init();
  
  Serial.println("RFID Door Strike System Ready. Scan card...");
}

void loop() {
  // Check for new cards
  if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
    
    bool isAuthorized = true;
    for (byte i = 0; i < 4; i++) {
      if (rfid.uid.uidByte[i] != MASTER_UID[i]) {
        isAuthorized = false;
        break;
      }
    }
    
    if (isAuthorized) {
      Serial.println("[SUCCESS] Authorized Card Scanned! Unlocking Strike...");
      
      // Trigger relay ON (closes NO contact, unlocking door)
      digitalWrite(RELAY_PIN, HIGH);
      
      // Hold door unlocked for 3 seconds
      delay(3000);
      
      // Turn relay OFF (relocking door)
      Serial.println("Relocking Door Strike...");
      digitalWrite(RELAY_PIN, LOW);
    } else {
      Serial.println("[DENIED] Card not authorized.");
      // Ensure lock is closed
      digitalWrite(RELAY_PIN, LOW);
    }
    
    rfid.PICC_HaltA();
    rfid.PCD_StopCrypto1();
  }
  
  delay(100);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **RFID Reader (MFRC522)**, and **Relay Module** onto the canvas.
2. Connect MFRC522: **SDA** to **D10**, **SCK** to **D13**, **MOSI** to **D11**, **MISO** to **D12**, **RST** to **D9**, **VCC** to **5V**, and **GND** to **GND**.
3. Connect Relay: **IN** to **D4**, **VCC** to **5V**, and **GND** to **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Click the RFID Reader component to trigger a simulated card swipe.
8. Watch the Relay contacts switch closed, pause for 3 seconds, and click back open.

## Expected Output

Terminal:
```
RFID Door Strike System Ready. Scan card...
[SUCCESS] Authorized Card Scanned! Unlocking Strike...
Relocking Door Strike...
...
```

### Expected Canvas Behavior

| Swiped Card UID | Match Check | Pin D4 Output | Relay contact switch |
| --- | --- | --- | --- |
| DE AD BE EF | Match | HIGH (5V) for 3s -> LOW | Closes COM-NO contacts |
| Other | Mismatch | LOW (0V) | Remains open |

The relay indicator turns green and switches state on the canvas when the authorized card is swiped.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `digitalWrite(RELAY_PIN, HIGH)` | Applies 5V to the relay control board, energizing the coil magnet. |
| `delay(3000)` | Keeps the relay active (unlocked) for 3 seconds before automatically resetting the output pin to LOW to relock. |

## Hardware & Safety Concept: Door Strikes & Flyback Diodes
Professional electromagnetic lock coils (solenoids) store high magnetic energy.
- When the relay switches OFF, this magnetic field collapses instantly, generating a massive **reverse voltage spike** (hundreds of volts) that travels back through the wiring.
- To prevent this spike from frying the relay contacts or resetting the Arduino, you must wire a **flyback diode** (like a 1N4007) in reverse parallel across the lock solenoid terminals. This safely routes the inductive feedback spike in a harmless loop.

## Try This! (Challenges)
1. **Indicator Lights**: Wire a Red LED on D7 and a Green LED on D8. Turn the Green LED ON only when the door is unlocked, and Red LED ON when locked.
2. **Access Log**: Print a message in the Terminal displaying how many times the lock has been successfully opened.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay clicks but door doesn't unlock | Lock wiring error | Ensure the door strike power supply is wired through the Relay's **COM** (Common) and **NO** (Normally Open) terminals in series. |
| Relay never clicks | Wrong control pin | Check that the Relay IN pin is wired to D4, matching `RELAY_PIN = 4`. |

## Mode Notes
These patterns (RFID logic checks triggering digital relay outputs) are supported by MbedO interpreted mode.

## Related Projects
- [39 - Auto Lamp Relay](39-auto-lamp-relay.md)
- [68 - RFID Access LED](68-rfid-access-led.md)
- [70 - RFID Servo Unlock](70-rfid-servo-unlock.md)
