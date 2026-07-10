# 141 - Pico RFID OLED

Build an RFID card scanner that displays personalized credentials and access verification status on a graphical OLED screen.

## Goal
Learn how to parse RFID card UIDs, match them against user memory coordinates, and draw custom message frames on SSD1306 OLED displays.

## What You Will Build
An office check-in OLED terminal:
- **MFRC522 RFID Reader**: Reads cards over SPI.
- **SSD1306 OLED (GP4, GP5)**: Displays checks (e.g. "ACCESS APPROVED" on a white background or "UNKNOWN CARD" warning grids).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MFRC522 | SDA / SCK / MOSI / MISO / RST | GP17 / GP18 / GP19 / GP16 / GP22 | SPI Bus |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int RST_PIN = 22;
const int SS_PIN  = 17;

MFRC522 mfrc522(SS_PIN, RST_PIN);

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

const byte authUID[4] = {0xDE, 0x12, 0xA4, 0xC3};

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  SPI.begin();
  mfrc522.PCD_Init();

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }
  
  showScanPrompt();
}

void loop() {
  if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
    bool match = true;
    if (mfrc522.uid.uidByte[0] != authUID[0]) { match = false; }
    if (mfrc522.uid.uidByte[1] != authUID[1]) { match = false; }
    if (mfrc522.uid.uidByte[2] != authUID[2]) { match = false; }
    if (mfrc522.uid.uidByte[3] != authUID[3]) { match = false; }

    display.clearDisplay();

    if (match) {
      // Access Granted screen (Inverted background)
      display.fillRect(0, 0, 128, 64, SSD1306_WHITE);
      display.setTextColor(SSD1306_BLACK);
      display.setTextSize(2);
      display.setCursor(10, 10);
      display.print("APPROVED");
      
      display.setTextSize(1);
      display.setCursor(10, 42);
      display.print("User: Alice");
    } else {
      // Access Denied screen
      display.setTextColor(SSD1306_WHITE);
      display.setTextSize(2);
      display.setCursor(15, 10);
      display.print("DENIED!");
      
      display.setTextSize(1);
      display.setCursor(15, 42);
      display.print("Unknown keycard");
      display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    }

    display.display();
    delay(2500); // Hold check-in log for 2.5 seconds
    
    showScanPrompt();
    mfrc522.PICC_HaltA();
  }

  delay(50);
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
1. Drag the **Raspberry Pi Pico**, **MFRC522 RFID Reader**, and **SSD1306 OLED** onto the canvas.
2. Connect MFRC522 SPI pins: **SDA** to **GP17**, **SCK** to **GP18**, **MOSI** to **GP19**, **MISO** to **GP16**, **RST** to **GP22**. Connect OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Scan authorized and unauthorized tags on canvas and watch the OLED verify credentials.

## Expected Output

Terminal:
```
Simulation active. OLED access log console online.
```

## Expected Canvas Behavior
* Startup: OLED shows a border frame reading `Scan card...`.
* Scan Card 1 (`DE 12 A4 C3`): Screen fills white, displaying `APPROVED` / `User: Alice` in black text.
* Scan Unknown Card: Screen shows `DENIED!` / `Unknown keycard` inside a border frame.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `display.fillRect(0, 0, 128, 64, SSD1306_WHITE)` | Inverts the screen to solid white to show high-priority success status. |

## Hardware & Safety Concept: User Feedback Design
In physical terminals (check-in counters, entry turnstiles), the screen is the primary interface for users. Displays must use high-contrast layouts (like inverted text blocks) and large text labels so the check status is readable from a distance.

## Try This! (Challenges)
1. **Audio Verification**: Connect a buzzer on GP14 and play success double-beeps or failure low buzzes.
2. **Access Relay Link**: Connect a relay on GP10 and activate it for 3 seconds on approved scans.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| OLED doesn't clear after a scan | Delay loops | Ensure `mfrc522.PICC_HaltA()` is called after scanning to prevent repeated reads from stalling the main loop. |

## Mode Notes
This multi-device I2C and SPI project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [85 - Pico RFID LED](../intermediate/85-pico-rfid-led.md)
- [111 - Pico RFID LCD](../intermediate/111-pico-rfid-lcd.md)
