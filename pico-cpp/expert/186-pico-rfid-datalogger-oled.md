# 186 - Pico RFID Datalogger OLED

Build an RFID office entry terminal that unlocks a door strike relay, displays check-in details on an OLED, and streams scan records over Bluetooth.

## Goal
Learn how to interface SPI RFID readers, actuate electronic locks, design high-contrast OLED check-in screens, and stream logs over Bluetooth.

## What You Will Build
A wireless office check-in gateway:
- **MFRC522 RFID Reader**: Reads user cards over SPI.
- **Relay Module (GP10)**: Controls the door latch strike.
- **SSD1306 OLED (GP4, GP5)**: Displays checks and greetings.
- **HC-05 Bluetooth Module (GP0, GP1)**: Transmits wireless check-in logs.
- **Bluetooth Stream**: Logs checks in CSV formats (e.g. `Card_UID,Status_code`).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MFRC522 | SDA / SCK / MOSI / MISO / RST | GP17 / GP18 / GP19 / GP16 / GP22 | SPI Bus |
| Relay Module | IN | GP10 | Door strike latch |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 lines |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int RST_PIN   = 22;
const int SS_PIN    = 17;
const int RELAY_PIN = 10;

MFRC522 mfrc522(SS_PIN, RST_PIN);

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

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

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }

  showScanPrompt();
  Serial1.println("OLED Access Gateway Online.");
}

void loop() {
  if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
    bool match = true;
    if (mfrc522.uid.uidByte[0] != authUID[0]) { match = false; }
    if (mfrc522.uid.uidByte[1] != authUID[1]) { match = false; }
    if (mfrc522.uid.uidByte[2] != authUID[2]) { match = false; }
    if (mfrc522.uid.uidByte[3] != authUID[3]) { match = false; }

    display.clearDisplay();

    // Print scanned card UID to Bluetooth
    Serial1.print("UID:");
    for (byte i = 0; i < mfrc522.uid.size; i++) {
      Serial1.print(mfrc522.uid.uidByte[i] < 0x10 ? " 0" : " ");
      Serial1.print(mfrc522.uid.uidByte[i], HEX);
    }

    if (match) {
      // Access Granted Screen (Inverted Colors)
      display.fillRect(0, 0, 128, 64, SSD1306_WHITE);
      display.setTextColor(SSD1306_BLACK);
      display.setTextSize(2);
      display.setCursor(10, 10);
      display.print("APPROVED");
      
      display.setTextSize(1);
      display.setCursor(10, 42);
      display.print("Welcome, Alice!");
      display.display();

      digitalWrite(RELAY_PIN, HIGH); // Open door strike
      Serial1.println(",STATUS:APPROVED");
      delay(3000); // Keep open for 3 seconds
      digitalWrite(RELAY_PIN, LOW);  // Lock door strike
    } else {
      // Access Denied Screen
      display.setTextColor(SSD1306_WHITE);
      display.setTextSize(2);
      display.setCursor(15, 10);
      display.print("DENIED!");
      
      display.setTextSize(1);
      display.setCursor(15, 42);
      display.print("Unknown keycard");
      display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
      display.display();

      Serial1.println(",STATUS:DENIED");
      delay(2000);
    }

    showScanPrompt();
    mfrc522.PICC_HaltA();
  }
  delay(100);
}

void showScanPrompt() {
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);
  display.setCursor(20, 15);
  display.print("Access Control");
  display.setCursor(20, 35);
  display.print("Scan card...");
  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.display();
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MFRC522 RFID Reader**, **Relay**, **SSD1306 OLED**, and **HC-05** onto the canvas.
2. Connect MFRC522 SPI pins, Relay to **GP10**, OLED to **GP4/GP5**, and HC-05 to **GP0/GP1**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Scan the authorized tag (`DE 12 A4 C3`) and watch the relay open and the log print over Bluetooth.

## Expected Output

Terminal (Bluetooth):
```
OLED Access Gateway Online.
UID: DE 12 A4 C3,STATUS:APPROVED
UID: AA BB CC DD,STATUS:DENIED
```

## Expected Canvas Behavior
* Startup: OLED shows `Scan card...`. Relay is OFF.
* Scan Card 1 (`DE 12 A4 C3`): Screen fills white displaying `APPROVED` / `Welcome, Alice!`, Relay turns ON for 3 seconds.
* Scan Card 2: Screen displays `DENIED!` / `Unknown keycard`, Relay remains OFF.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `display.fillRect(0, 0, 128, 64, SSD1306_WHITE)` | Inverts the screen to solid white to show high-priority success status. |

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
- [111 - Pico RFID LCD](../intermediate/111-pico-rfid-lcd.md)
- [141 - Pico RFID OLED](../advanced/141-pico-rfid-oled.md)
- [160 - Pico Smart Lock EEPROM](../advanced/160-pico-smart-lock-eeprom.md)
- [174 - Pico Access Control Datalogger](174-pico-access-control-datalogger.md)
