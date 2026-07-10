# 174 - Pico Access Control Datalogger

Build an RFID office entry terminal that unlocks a door strike relay and streams check-in log records wirelessly over Bluetooth.

## Goal
Learn how to interface SPI RFID readers, actuate solenoid strike door relays, update status LCDs, and transmit formatted serial data packets over Bluetooth.

## What You Will Build
A wireless office check-in portal:
- **MFRC522 RFID Reader**: Reads user cards over SPI.
- **Relay Module (GP10)**: Controls the electronic door latch.
- **HC-05 Bluetooth Module (GP0, GP1)**: Streams access logs wirelessly.
- **16x2 I2C LCD (GP4, GP5)**: Displays greetings and status logs.
- **Bluetooth Stream**: Logs checks in CSV-like formats (e.g. `Card_UID,Access_status`).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MFRC522 | SDA / SCK / MOSI / MISO / RST | GP17 / GP18 / GP19 / GP16 / GP22 | SPI Bus |
| Relay Module | IN | GP10 | Door strike latch |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 lines |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground return |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int RST_PIN   = 22;
const int SS_PIN    = 17;
const int RELAY_PIN = 10;

MFRC522 mfrc522(SS_PIN, RST_PIN);
LiquidCrystal_I2C lcd(0x27, 16, 2);

const byte authUID[4] = {0xDE, 0x12, 0xA4, 0xC3};

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  SPI.begin();
  mfrc522.PCD_Init();

  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start locked

  lcd.init();
  lcd.backlight();
  showLockedScreen();

  Serial1.println("Office Access Gateway Online.");
}

void loop() {
  if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
    bool match = true;
    if (mfrc522.uid.uidByte[0] != authUID[0]) { match = false; }
    if (mfrc522.uid.uidByte[1] != authUID[1]) { match = false; }
    if (mfrc522.uid.uidByte[2] != authUID[2]) { match = false; }
    if (mfrc522.uid.uidByte[3] != authUID[3]) { match = false; }

    lcd.clear();
    
    // Print scanned card UID to Bluetooth
    Serial1.print("UID:");
    for (byte i = 0; i < mfrc522.uid.size; i++) {
      Serial1.print(mfrc522.uid.uidByte[i] < 0x10 ? " 0" : " ");
      Serial1.print(mfrc522.uid.uidByte[i], HEX);
    }

    if (match) {
      // Access Approved
      lcd.setCursor(0, 0);
      lcd.print("Welcome, Alice! ");
      lcd.setCursor(0, 1);
      lcd.print("Door Unlocked   ");
      
      digitalWrite(RELAY_PIN, HIGH); // Open door strike
      Serial1.println(",STATUS:APPROVED");
      delay(3000); // Keep open for 3 seconds
      digitalWrite(RELAY_PIN, LOW);  // Lock door strike
    } else {
      // Access Denied
      lcd.setCursor(0, 0);
      lcd.print("Access Denied!  ");
      lcd.setCursor(0, 1);
      lcd.print("Unknown keycard ");
      Serial1.println(",STATUS:DENIED");
      delay(2000);
    }

    showLockedScreen();
    mfrc522.PICC_HaltA();
  }
  delay(100);
}

void showLockedScreen() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Access Console  ");
  lcd.setCursor(0, 1);
  lcd.print("Scan card...    ");
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MFRC522 RFID Reader**, **Relay**, **HC-05**, and **I2C LCD** onto the canvas.
2. Connect MFRC522 SPI pins, Relay to **GP10**, HC-05 to **GP0/GP1**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Scan the authorized tag (`DE 12 A4 C3`) and watch the relay open and the log print over Bluetooth.

## Expected Output

Terminal (Bluetooth):
```
Office Access Gateway Online.
UID: DE 12 A4 C3,STATUS:APPROVED
UID: AA BB CC DD,STATUS:DENIED
```

## Expected Canvas Behavior
* Startup: LCD reads `Scan card...`. Relay is OFF.
* Scan Card 1 (`DE 12 A4 C3`): LCD reads `Welcome, Alice!`, Relay turns ON (GP10 HIGH) for 3 seconds.
* Scan Card 2: LCD reads `Access Denied!`, Relay remains OFF.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Serial1.print(..., HEX)` | Formats and prints the scanned card's UID bytes as a hexadecimal string over the Bluetooth serial channel. |

## Hardware & Safety Concept: Door Strike Power Isolation
Dynamic security locks use solenoid electronic door strikes or high-torque servos to lock/unlock doors. Solenoids draw high currents and generate magnetic back-EMF spikes when turned OFF. To protect the Pico, use a optocoupler-isolated relay module to control the strike, and power it from a separate external power block, not the Pico's 3.3V pin.

## Try This! (Challenges)
1. **Auditory Alarm**: Connect a buzzer on GP14 and sound a double beep on approved cards, and a long low tone on denied cards.
2. **Lockout Latch**: Disable scans for 2 minutes if an invalid card is scanned 3 times in a row.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD flashes repeatedly | Read loop logic | Make sure to call `mfrc522.PICC_HaltA()` after checking a card to prevent repeated reads from resetting the LCD. |

## Mode Notes
This multi-device I2C and SPI project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [87 - Pico RFID Servo](../intermediate/87-pico-rfid-servo.md)
- [111 - Pico RFID LCD](../intermediate/111-pico-rfid-lcd.md)
- [141 - Pico RFID OLED](../advanced/141-pico-rfid-oled.md)
- [160 - Pico Smart Lock EEPROM](../advanced/160-pico-smart-lock-eeprom.md)
