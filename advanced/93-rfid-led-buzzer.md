# 93 - RFID LED Buzzer

Extend the basic RFID reader to drive a three-LED traffic light (green/yellow/red) and a buzzer chime pattern based on whether the scanned card UID matches a pre-programmed authorized list.

## Goal
Learn how to maintain an authorized UID list in firmware, perform a multi-byte comparison, and coordinate multi-output responses (LED + buzzer) to communicate access decision results clearly.

## What You Will Build
When a card is scanned:
- **Authorized card**: Green LED lights, buzzer plays two quick ascending beeps, and "ACCESS GRANTED" prints to the Terminal.
- **Unauthorized card**: Red LED lights, buzzer plays one long low tone, and "ACCESS DENIED" prints to the Terminal.
- A brief yellow LED flash acknowledges every scan attempt regardless of result.

**Why D2–D8 and SPI pins?** The RC522 RFID reader uses the SPI bus (D10 CS, D11 MOSI, D12 MISO, D13 SCK) plus D9 RST. Pins D3, D4, D5 drive the LEDs. Pin D8 drives the buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| RC522 RFID Reader | `rc522_rfid` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| LED (Yellow) | `led` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes (per LED) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| RC522 RFID | SDA (CS) | D10 | Chip select (SPI) |
| RC522 RFID | SCK | D13 | SPI Clock |
| RC522 RFID | MOSI | D11 | SPI Data out |
| RC522 RFID | MISO | D12 | SPI Data in |
| RC522 RFID | RST | D9 | Reset signal |
| RC522 RFID | 3.3V | 3.3V | Power supply |
| RC522 RFID | GND | GND | Ground reference |
| Green LED | Anode (+) | D3 | Access granted indicator |
| Green LED | Cathode (-) | GND | Via 220 ohm resistor |
| Yellow LED | Anode (+) | D4 | Scan detected indicator |
| Yellow LED | Cathode (-) | GND | Via 220 ohm resistor |
| Red LED | Anode (+) | D5 | Access denied indicator |
| Red LED | Cathode (-) | GND | Via 220 ohm resistor |
| Buzzer | + | D8 | Signal pin |
| Buzzer | - | GND | Ground reference |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>

const int RST_PIN   = 9;
const int SS_PIN    = 10;
const int GREEN_PIN = 3;
const int YEL_PIN   = 4;
const int RED_PIN   = 5;
const int BUZ_PIN   = 8;

MFRC522 rfid(SS_PIN, RST_PIN);

// Authorized UID list - add your card UIDs here
// Each entry is a 4-byte array matching the RC522 UID bytes
byte authorizedUIDs[][4] = {
  {0xDE, 0xAD, 0xBE, 0xEF},
  {0x01, 0x23, 0x45, 0x67}
};
const int AUTH_COUNT = sizeof(authorizedUIDs) / sizeof(authorizedUIDs[0]);

void allLEDsOff() {
  digitalWrite(GREEN_PIN, LOW);
  digitalWrite(YEL_PIN,   LOW);
  digitalWrite(RED_PIN,   LOW);
}

bool isAuthorized(byte *uid, byte uidSize) {
  if (uidSize != 4) return false;
  for (int i = 0; i < AUTH_COUNT; i++) {
    bool match = true;
    for (int j = 0; j < 4; j++) {
      if (uid[j] != authorizedUIDs[i][j]) { match = false; break; }
    }
    if (match) return true;
  }
  return false;
}

void setup() {
  Serial.begin(9600);
  SPI.begin();
  rfid.PCD_Init();
  
  pinMode(GREEN_PIN, OUTPUT);
  pinMode(YEL_PIN,   OUTPUT);
  pinMode(RED_PIN,   OUTPUT);
  pinMode(BUZ_PIN,   OUTPUT);
  allLEDsOff();
  
  Serial.println("RFID LED Buzzer System Ready. Scan a card...");
}

void loop() {
  if (!rfid.PICC_IsNewCardPresent() || !rfid.PICC_ReadCardSerial()) return;
  
  // Flash yellow on any scan
  digitalWrite(YEL_PIN, HIGH);
  delay(80);
  digitalWrite(YEL_PIN, LOW);
  
  // Print UID
  Serial.print("Card UID: ");
  for (byte i = 0; i < rfid.uid.size; i++) {
    Serial.print(rfid.uid.uidByte[i] < 0x10 ? "0" : "");
    Serial.print(rfid.uid.uidByte[i], HEX);
    Serial.print(" ");
  }
  Serial.println();
  
  if (isAuthorized(rfid.uid.uidByte, rfid.uid.size)) {
    Serial.println("-> ACCESS GRANTED");
    digitalWrite(GREEN_PIN, HIGH);
    tone(BUZ_PIN, 1000, 120); delay(200);
    tone(BUZ_PIN, 1400, 120); delay(300);
    noTone(BUZ_PIN);
    digitalWrite(GREEN_PIN, LOW);
  } else {
    Serial.println("-> ACCESS DENIED");
    digitalWrite(RED_PIN, HIGH);
    tone(BUZ_PIN, 350, 600);
    delay(700);
    noTone(BUZ_PIN);
    digitalWrite(RED_PIN, LOW);
  }
  
  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
  
  delay(500); // Cooldown between scans
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **RC522 RFID Reader**, **three LEDs** (Green, Yellow, Red), and **Buzzer** onto the canvas.
2. Wire RC522 per the wiring table above.
3. Connect LEDs to **D3**, **D4**, **D5** (with cathodes to GND).
4. Connect Buzzer **+** to **D8**, **-** to **GND**.
5. Paste the code into the editor.
6. Use the default Arduino interpreted runtime.
7. Click **Run**.
8. Click the RFID reader on the canvas to simulate card scans. Observe LED and Terminal responses.

## Expected Output

Terminal (Authorized card):
```
Card UID: DE AD BE EF
-> ACCESS GRANTED
```

Terminal (Unknown card):
```
Card UID: AA BB CC DD
-> ACCESS DENIED
```

## Expected Canvas Behavior

| Scanned UID | Match | Yellow Flash | Green/Red LED | Buzzer Pattern |
| --- | --- | --- | --- | --- |
| `DE AD BE EF` | Yes | Brief flash | Green 300 ms | Two ascending beeps |
| `AA BB CC DD` | No | Brief flash | Red 700 ms | One long low tone |

## Code Walkthrough

| Expression | What It Does |
| --- | --- |
| `authorizedUIDs[][4]` | 2D byte array storing one 4-byte UID per row. Extend by adding more rows. |
| `isAuthorized()` | Iterates every stored UID and compares byte-by-byte. Returns `true` on first match found. |
| `sizeof(authorizedUIDs) / sizeof(authorizedUIDs[0])` | Automatically calculates the number of UIDs in the array without hardcoding a count. |

## Hardware & Safety Concept: UID Spoofing Risk
Standard MFRC522-based systems use the 4-byte PICC UID as the sole credential. This is a known weakness: cheap UID-writable "magic" cards can be programmed to clone any UID.
- For production security, use the MIFARE authentication (sector key A/B) to verify cryptographic sector access rather than relying on UID alone.
- For educational and hobbyist projects, UID matching is a useful and clearly understandable starting point.

## Try This! (Challenges)
1. **Add More Cards**: Extend the `authorizedUIDs` array with a third row. Scan a new card, copy its printed UID from the Terminal, and enter it in hex.
2. **Access Log Counter**: Add a global `int accessCount = 0` variable. Increment it on every grant and print the cumulative count after each decision.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Card never detected | SPI wiring incorrect | Verify all five SPI lines: CS to D10, MOSI to D11, MISO to D12, SCK to D13, RST to D9. |
| Every card shows "DENIED" | UID in array does not match | Scan your actual card, read the printed UID from Terminal, and update the array. |

## Mode Notes
These patterns (RFID UID read, multi-output coordination) are supported by MbedO interpreted mode.

## Related Projects
- [67 - RFID UID Print](../intermediate/67-rfid-uid-print.md)
- [68 - RFID Access LED](../intermediate/68-rfid-access-led.md)
- [69 - RFID Access Buzzer](../intermediate/69-rfid-access-buzzer.md)
- [94 - RFID Servo LCD](94-rfid-servo-lcd.md)
