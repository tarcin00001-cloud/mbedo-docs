# 37 - ESP32 LCD Network Clock

Configure the ESP32 to connect to a local WiFi network, sync its internal Real-Time Clock (RTC) using Network Time Protocol (NTP) servers, and display the formatted date and time on a 16x2 I2C LCD.

## Goal
Learn how to display network time on a 16x2 LCD, format date-time strings, manage I2C displays, and blink indicators.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A 16x2 I2C LCD is on I2C (GPIO 21/22). The ESP32 syncs with an NTP server and displays the local date (Row 0) and time (Row 1), blinking the colon indicator every second.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the I2C bus. Power the LCD from the 5V Vin rail.

## Code
```cpp
// LCD Network Clock (NTP synced + 16x2 LCD)
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <WiFi.h>
#include "time.h"

LiquidCrystal_I2C lcd(0x27, 16, 2);

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// NTP Configuration (IST UTC+5:30 = 19800 seconds)
const char* ntpServer = "pool.ntp.org";
const long  gmtOffset_sec = 19800; 
const int   daylightOffset_sec = 0;

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Network Clock");
  lcd.setCursor(0, 1);
  lcd.print("Connecting WiFi");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    lcd.print(".");
  }
  
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Syncing Time...");
  
  // Initialize NTP time synchronization
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  
  struct tm timeinfo;
  while (!getLocalTime(&timeinfo)) {
    delay(500);
    lcd.print(".");
  }
  
  lcd.clear();
  lcd.print("Clock Active!");
  delay(1500);
  lcd.clear();
}

void loop() {
  struct tm timeinfo;
  
  if (getLocalTime(&timeinfo)) {
    // 1. Format Date String (Row 0: e.g. "Date: 11/07/2026")
    char dateStr[16];
    strftime(dateStr, sizeof(dateStr), "Date: %d/%m/%Y", &timeinfo);
    lcd.setCursor(0, 0);
    lcd.print(dateStr);
    
    // 2. Format Time String (Row 1: e.g. "Time: 14:35:10")
    char timeStr[16];
    bool showColon = (timeinfo.tm_sec % 2 == 0);
    
    if (showColon) {
      // Show colon
      strftime(timeStr, sizeof(timeStr), "Time: %H:%M:%S", &timeinfo);
    } else {
      // Hide colon (blink effect)
      strftime(timeStr, sizeof(timeStr), "Time: %H %M %S", &timeinfo);
    }
    
    lcd.setCursor(0, 1);
    lcd.print(timeStr);
  }
  
  // Refresh display every 500 ms for smooth colon blinking
  delay(500);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **16x2 I2C LCD** onto the canvas.
2. Wire LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Watch the LCD screen display the local time with a blinking colon.

## Expected Output
LCD Display:
```
Date: 11/07/2026
Time: 06:50:02  (Time characters with blinking colon)
```

## Expected Canvas Behavior
* The LCD display initializes and updates the local date and time every 500 ms.
* The colons in the time string blink.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `configTime(...)` | Configures the local time zone offset and triggers background NTP synchronization. |
| `timeinfo.tm_sec % 2 == 0` | Checks if the second counter is even to blink the colon. |
| `strftime(...)` | Formats date-time structures into strings. |

## Hardware & Safety Concept: Network Time Synchronization (NTP)
Microcontrollers lack internal battery-backed Real-Time Clocks (RTCs), so their internal time resets on reboot. The **NTP (Network Time Protocol)** syncs the local clock with atomic time servers over the network. Once synchronized, the ESP32's internal RTC maintains the time in the background, allowing scheduling tasks without manual adjustments.

## Try This! (Challenges)
1. **Alarm Clock**: Add a buzzer (GPIO 4) and sound an alarm at a set time.
2. **Format Switcher**: Add a button on GPIO 12 to toggle between 12-hour (AM/PM) and 24-hour formats.
3. **Temperature Display**: Add a DHT22 sensor (Project 100) and display the temperature alongside the time.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD display freezes | I2C address mismatch | Verify the I2C address in `LiquidCrystal_I2C` matches your LCD module (usually `0x27`) |
| Time does not update | NTP sync failed | Verify that the ESP32 is connected to the network before initializing NTP |
| Time is offset by 1 hour | DST setting incorrect | Verify that the DST offset parameter matches the local season |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [34 - ESP32 NTP Time Server sync](34-esp32-ntp-time-server-sync.md)
- [35 - ESP32 NTP Time Zone offset adjustments](35-esp32-ntp-time-zone-offset-adjustments.md)
- [36 - ESP32 OLED Digital Clock (NTP synced)](36-esp32-oled-digital-clock-ntp-synced.md)
