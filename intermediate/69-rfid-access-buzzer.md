# 69 - RFID Access Buzzer

Add audible success chimes and error alarms to an RFID access control system using a buzzer.

## Goal
Learn how to use positive and negative auditory feedback frequencies to signal system access states to users.

## What You Will Build
When a card is swiped:
- If the card UID matches the master key (`DE AD BE EF`), the buzzer connected to pin D6 plays a high-pitched "success" chime (double beep: 1500 Hz then 2000 Hz).
- If it does not match, the buzzer sounds a low-pitch "error" buzz (200 Hz for 600 ms).

**Why D6?** Pin D6 drives the audio frequencies on the warning buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| 100 ohm resistor | `resistor` | Optional | Yes |

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
| Buzzer | Pin 1 (Positive) | D6 | Audio control line |
| Buzzer | Pin 2 (Negative) | GND | Ground reference |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>

#define SS_PIN 10
#define RST_PIN 9
#define BUZZ_PIN 6

MFRC522 rfid(SS_PIN, RST_PIN);

// Authorized Master Card UID
const byte MASTER_UID[] = {0xDE, 0xAD, 0xBE, 0xEF};

void setup() {
  pinMode(BUZZ_PIN, OUTPUT);
  Serial.begin(9600);
  SPI.begin();
  rfid.PCD_Init();
  
  Serial.println("Audio Access System Ready. Scan card...");
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
      Serial.println("[SUCCESS] Access Approved!");
      
      // Play Happy Chime: two short rising notes
      tone(BUZZ_PIN, 1500);
      delay(80);
      tone(BUZZ_PIN, 2000);
      delay(150);
      noTone(BUZZ_PIN);
    } else {
      Serial.println("[DENIED] Access Blocked!");
      
      // Play Error Buzz: low pitch buzz
      tone(BUZZ_PIN, 200);
      delay(600);
      noTone(BUZZ_PIN);
    }
    
    rfid.PICC_HaltA();
    rfid.PCD_StopCrypto1();
  }
  
  delay(100);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **RFID Reader (MFRC522)**, and **Buzzer** onto the canvas.
2. Connect MFRC522: **SDA** to **D10**, **SCK** to **D13**, **MOSI** to **D11**, **MISO** to **D12**, **RST** to **D9**, **VCC** to **5V**, and **GND** to **GND**.
3. Connect Buzzer **pin 1** to Arduino **D6** and **pin 2** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Click the RFID Reader component to trigger a simulated card swipe.
8. Listen to the high-pitched success chime.

## Expected Output

Terminal:
```
Audio Access System Ready. Scan card...
[SUCCESS] Access Approved!
...
```

### Expected Canvas Behavior

| Swiped Card UID | Match Check | Buzzer State (D6) | Acoustic Chime |
| --- | --- | --- | --- |
| DE AD BE EF | Match | Pulsed HIGH (1500/2000 Hz) | Success Double Beep |
| Other | Mismatch | Solid HIGH (200 Hz) | Long Error Buzz (600 ms) |

The buzzer vibrates on the canvas to play the respective audio chime depending on the card validity.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `tone(BUZZ_PIN, 1500); delay(80); tone(BUZZ_PIN, 2000);` | Generates a quick rising double tone sequence (note F6 followed by B6) indicating success. |
| `tone(BUZZ_PIN, 200); delay(600);` | Generates a continuous low-frequency hum (200 Hz, near G3) indicating access rejection. |

## Hardware & Safety Concept: Accessibility in Hardware UI
Physical access control terminals must provide multi-sensory feedback. 
- Visual feedback (LEDs) helps users in noisy areas or who have hearing impairments.
- Auditory feedback (Buzzers) helps users who are visually impaired or who are not looking directly at the scanner faceplate when swiping, ensuring they receive immediate feedback on system state.

## Try This! (Challenges)
1. **Siren Denied Alert**: Modify the error block to generate a siren wail (alternating between 800 Hz and 1600 Hz three times) if an unauthorized card is swiped.
2. **Shorten Beeps**: Make the chimes shorter and sharper to conserve system power and reduce noise.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer hums constantly | Program stuck in tone loop | Ensure `noTone(BUZZ_PIN)` is called at the end of both logic branches. |
| No sound | Ground missing or wrong pin | Verify that the buzzer pin connects to D6, matching `BUZZ_PIN = 6`. |

## Mode Notes
These patterns (scanned RFID comparison triggering tone arrays and lockout delays) are supported by MbedO interpreted mode.

## Related Projects
- [67 - RFID UID Print](67-rfid-uid-print.md)
- [68 - RFID Access LED](68-rfid-access-led.md)
- [70 - RFID Servo Unlock](70-rfid-servo-unlock.md)
