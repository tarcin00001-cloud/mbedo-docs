# 36 - ESP32 OLED Digital Clock (NTP synced)

Build a network-synchronized digital desk clock on the ESP32 that connects to a local WiFi network, syncs its clock with NTP servers, and displays the formatted local date and time on an SSD1306 OLED screen with a blinking colon indicator.

## Goal
Learn how to display network time on OLED screens, format time strings, manage screen update rates, and sync clocks.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An SSD1306 OLED is on I2C (GPIO 21/22). The ESP32 syncs with an NTP server and displays the local time in large text, blinking the colon indicator every second.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 0.96″ SSD1306 OLED (I2C) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| SSD1306 OLED | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the I2C bus. Confirm the OLED address is correct (usually `0x3C`).

## Code
```cpp
// OLED Digital Clock (NTP synced + Blinking Colon)
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <WiFi.h>
#include "time.h"

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
#define OLED_ADDR     0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// NTP Configuration (IST UTC+5:30 = 19800 seconds)
const char* ntpServer = "pool.ntp.org";
const long  gmtOffset_sec = 19800; 
const int   daylightOffset_sec = 0;

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED allocation failed!");
    while(1) {}
  }
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.println("OLED Digital Clock");
  display.println("Connecting WiFi...");
  display.display();
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    display.print(".");
    display.display();
  }
  
  display.println("\nWiFi Connected.");
  display.println("Syncing NTP Time...");
  display.display();
  
  // Initialize NTP time synchronization
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  
  struct tm timeinfo;
  while (!getLocalTime(&timeinfo)) {
    delay(500);
    display.print(".");
    display.display();
  }
  
  display.println("\nTime Synced!");
  display.display();
  delay(1000);
  display.clearDisplay();
}

void loop() {
  struct tm timeinfo;
  
  if (getLocalTime(&timeinfo)) {
    display.clearDisplay();
    
    // 1. Draw Header / Status
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 0);
    display.print("NTP DIGITAL CLOCK");
    display.drawFastHLine(0, 10, 128, SSD1306_WHITE);
    
    // 2. Format Date String (e.g. Saturday, Jul 11)
    char dateStr[32];
    strftime(dateStr, sizeof(dateStr), "%A, %b %d", &timeinfo);
    display.setCursor(0, 18);
    display.print(dateStr);
    
    // 3. Format Time String (Blink colon every second)
    char timeStr[16];
    bool showColon = (timeinfo.tm_sec % 2 == 0);
    
    if (showColon) {
      // Show colon: "14:35:10"
      strftime(timeStr, sizeof(timeStr), "%H:%M:%S", &timeinfo);
    } else {
      // Hide colon: "14 35 10" (blink effect)
      strftime(timeStr, sizeof(timeStr), "%H %M %S", &timeinfo);
    }
    
    // 4. Render Time in large text
    display.setTextSize(2);
    display.setCursor(0, 36);
    display.print(timeStr);
    
    display.display();
  }
  
  // Refresh display every 500 ms for smooth colon blinking
  delay(500);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **SSD1306 OLED** onto the canvas.
2. Wire OLED to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Watch the OLED screen display the local time with a blinking colon.

## Expected Output
OLED Display Layout:
```
NTP DIGITAL CLOCK
───────────────────
Saturday, Jul 11

06:50:02  (Time characters with blinking colon)
```

## Expected Canvas Behavior
* The OLED display initializes and updates the local date and time every 500 ms.
* The colons in the time string blink.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `configTime(...)` | Configures the local time zone offset and triggers background NTP synchronization. |
| `timeinfo.tm_sec % 2 == 0` | Checks if the second counter is even to blink the colon. |
| `strftime(...)` | Formats date-time structures into strings. |
| `display.display()` | Renders the compiled frame buffer to the OLED screen. |

## Hardware & Safety Concept: Refresh Rates and Screen Burn-In
OLED screens can experience screen burn-in (where static elements leave a permanent ghost image) if active for long periods. To protect the screen:
1. **Blinking Elements**: Blinking the colon indicator changes the state of the pixels, reducing localized burn-in.
2. **Periodic sleep**: Configure the OLED to turn off during nighttime hours when not in use.

## Try This! (Challenges)
1. **Alarm Clock**: Add a buzzer (GPIO 4) and sound an alarm at a set time.
2. **Format Switcher**: Add a button on GPIO 12 to toggle between 12-hour (AM/PM) and 24-hour formats.
3. **Temperature Display**: Add a DHT22 sensor (Project 100) and display the temperature alongside the time.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| OLED screen stays black | I2C address mismatch | Verify the I2C address in `display.begin()` matches your OLED module (usually `0x3C`) |
| Time does not update | NTP sync failed | Verify that the ESP32 is connected to the network before initializing NTP |
| Time is offset by 1 hour | DST setting incorrect | Verify that the DST offset parameter matches the local season |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [34 - ESP32 NTP Time Server sync](34-esp32-ntp-time-server-sync.md)
- [35 - ESP32 NTP Time Zone offset adjustments](35-esp32-ntp-time-zone-offset-adjustments.md)
- [37 - ESP32 LCD Network Clock](37-esp32-lcd-network-clock.md)
