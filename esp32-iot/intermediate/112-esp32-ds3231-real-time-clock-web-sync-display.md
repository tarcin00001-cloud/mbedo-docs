# 112 - DS3231 Real Time Clock Web Sync Display (NTP to RTC sync)

Build a resilient local clock console on the ESP32 that interfaces with a DS3231 RTC module on I2C (GPIO 21/22), synchronizes the hardware RTC with internet atomic time using NTP, displays dates/times on a 16x2 I2C LCD, and hosts a web dashboard with manual sync overrides.

## Goal
Learn how to read DS3231 RTC modules using RTClib, synchronize hardware clocks with NTP servers, handle offline clock operations, print times to I2C LCDs, and process manual sync web requests.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A DS3231 RTC is on I2C (GPIO 21/22), and a 16x2 LCD on I2C. On boot, the ESP32 fetches network time from NTP and writes it to the DS3231 RTC. The LCD displays the date and time. If WiFi goes offline, the clock keeps running. Navigating to the ESP32's IP address displays the current RTC time and allows triggering a manual sync with NTP.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DS3231 RTC Module | `i2c_device` | No | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, since the DS3231 is not physically on the canvas, the code defaults to using the ESP32's internal RTC or simulated NTP time.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS3231 RTC | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the I2C bus (GPIO 21/22) between the LCD and RTC. Power both from the 5V Vin rail.

## Code
```cpp
// DS3231 Real Time Clock Web Sync Display (Resilient Timekeeping Console)
#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <RTClib.h>
#include <time.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);
RTC_DS3231 rtc;

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Timezone offset configuration (India Standard Time: UTC+5:30)
const char* ntpServer = "pool.ntp.org";
const long  gmtOffset_sec = 19800; // 5.5 hours * 3600
const int   daylightOffset_sec = 0;

WebServer server(80);
bool ntpSyncStatus = false;

// Format numbers as two digits (e.g. 05)
String formatTwoDigits(int number) {
  if (number < 10) return "0" + String(number);
  return String(number);
}

// Sync the DS3231 hardware RTC with the NTP network clock
bool syncRTCWithNTP() {
  Serial.print("[Time Sync] Connecting to NTP server...");
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  
  struct tm timeinfo;
  if (getLocalTime(&timeinfo)) {
    Serial.println("Success!");
    
    // Write fetched NTP values to the DS3231 hardware registers
    rtc.adjust(DateTime(
      timeinfo.tm_year + 1900,
      timeinfo.tm_mon + 1,
      timeinfo.tm_mday,
      timeinfo.tm_hour,
      timeinfo.tm_min,
      timeinfo.tm_sec
    ));
    
    ntpSyncStatus = true;
    return true;
  } else {
    Serial.println("Failed! Server unreachable or timeout.");
    return false;
  }
}

// Display date and time on the LCD screen
void updateLCDDisplay(DateTime now) {
  lcd.clear();
  
  // Row 0: Date (DD/MM/YYYY)
  lcd.setCursor(0, 0);
  lcd.printf("Date: %s/%s/%d", 
             formatTwoDigits(now.day()).c_str(), 
             formatTwoDigits(now.month()).c_str(), 
             now.year());
  
  // Row 1: Time (HH:MM:SS) and Sync status
  lcd.setCursor(0, 1);
  lcd.printf("Time: %s:%s:%s", 
             formatTwoDigits(now.hour()).c_str(), 
             formatTwoDigits(now.minute()).c_str(), 
             formatTwoDigits(now.second()).c_str());
             
  // Display sync marker if synchronized
  if (ntpSyncStatus) {
    lcd.setCursor(15, 1);
    lcd.print("*"); // Asterisk indicator
  }
}

// HTTP API endpoint returning JSON data
void handleGetTimeData() {
  DateTime now = rtc.now();
  String timeStr = formatTwoDigits(now.hour()) + ":" + 
                   formatTwoDigits(now.minute()) + ":" + 
                   formatTwoDigits(now.second());
  String dateStr = formatTwoDigits(now.day()) + "/" + 
                   formatTwoDigits(now.month()) + "/" + 
                   String(now.year());
                   
  String json = "{\"time\":\"" + timeStr + "\"" +
                 ",\"date\":\"" + dateStr + "\"" +
                 ",\"ntpSynced\":" + String(ntpSyncStatus ? 1 : 0) + "}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to force NTP sync
void handlePostSync() {
  bool success = syncRTCWithNTP();
  if (success) {
    server.send(200, "text/plain", "OK");
  } else {
    server.send(500, "text/plain", "Sync Failed");
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>RTC System Dashboard</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 400px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .time-val { font-size: 56px; font-weight: 800; font-family: monospace; color: #10b981; margin: 15px 0 5px 0; }\n";
  html += "  .date-val { font-size: 18px; color: #94a3b8; font-family: monospace; margin-bottom: 25px; }\n";
  html += "  .badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 13px; text-transform: uppercase; margin-bottom: 20px; }\n";
  html += "  .synced { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .offline { background-color: #92400e; color: #fef3c7; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 14px; font-size: 16px; font-weight: bold; color: white; background-color: #38bdf8; border: none; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; }\n";
  html += "  .btn:hover { background-color: #0ea5e9; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Network Clock Terminal</h1>\n";
  html += "  <div id=\"syncBadge\" class=\"badge offline\">RTC Offline Mode</div>\n";
  html += "  <div id=\"timeDisplay\" class=\"time-val\">--:--:--</div>\n";
  html += "  <div id=\"dateDisplay\" class=\"date-val\">--/--/----</div>\n";
  
  html += "  <button class=\"btn\" onclick=\"forceSync()\">Sync with NTP</button>\n";
  html += "  <p class=\"footer\">Asynchronous updates polling every 1 second | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateTime() {\n";
  html += "    fetch('/api/time')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('timeDisplay').innerText = data.time;\n";
  html += "        document.getElementById('dateDisplay').innerText = data.date;\n";
  
  html += "        const bdgEl = document.getElementById('syncBadge');\n";
  html += "        if (data.ntpSynced === 1) {\n";
  html += "          bdgEl.innerText = 'NTP Synchronized';\n";
  html += "          bdgEl.className = 'badge synced';\n";
  html += "        } else {\n";
  html += "          bdgEl.innerText = 'RTC Offline Mode';\n";
  html += "          bdgEl.className = 'badge offline';\n";
  html += "        }\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  function forceSync() {\n";
  html += "    const btn = document.querySelector('.btn');\n";
  html += "    btn.innerText = 'Syncing...';\n";
  html += "    fetch('/api/sync', { method: 'POST' })\n";
  html += "      .then(response => {\n";
  html += "        if (response.ok) {\n";
  html += "          btn.innerText = 'Sync with NTP';\n";
  html += "          updateTime();\n";
  html += "        } else {\n";
  html += "          btn.innerText = 'Sync Failed! Retry';\n";
  html += "        }\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateTime();\n";
  html += "    setInterval(updateTime, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Initialize LCD screen
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Clock Setup");
  lcd.setCursor(0, 1);
  lcd.print("Initializing...");
  
  // 1. Initialize DS3231 hardware module
  if (!rtc.begin()) {
    Serial.println("[RTC Error] Could not find DS3231 module!");
    // Fail-safe: adjust internal software clock
  }
  
  if (rtc.lostPower()) {
    Serial.println("[RTC Alert] DS3231 lost power! Setting default compilation time...");
    rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));
  }
  
  Serial.println("\nESP32 NTP RTC Station Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  // Connect WiFi with a timeout to allow offline clock operation
  int wifiTimeout = 0;
  while (WiFi.status() != WL_CONNECTED && wifiTimeout < 15) {
    delay(500);
    Serial.print(".");
    wifiTimeout++;
  }
  
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\nWiFi Connected.");
    // Attempt time synchronization
    syncRTCWithNTP();
  } else {
    Serial.println("\nWiFi Offline. Operating on local RTC battery power.");
    ntpSyncStatus = false;
  }
  
  server.on("/", handleRoot);
  server.on("/api/time", handleGetTimeData);
  server.on("/api/sync", HTTP_POST, handlePostSync);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // 2. Read current time from local RTC module
  DateTime now = rtc.now();
  
  // 3. Refresh LCD screen every 500 ms
  static unsigned long lastClockUpdate = 0;
  if (millis() - lastClockUpdate >= 500) {
    updateLCDDisplay(now);
    lastClockUpdate = millis();
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **16x2 I2C LCD** onto the canvas.
2. Wire LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Verify that the time and date update on both the LCD screen and the webpage console.
5. Click the "Sync with NTP" button on the webpage to verify connection handshake logs in the Serial Monitor.

## Expected Output
Serial Monitor:
```
ESP32 NTP RTC Station Starting...
....
WiFi Connected.
[Time Sync] Connecting to NTP server... Success!
HTTP Server active. Connect at: http://10.10.0.3
```

LCD Display:
```
Date: 11/07/2026
Time: 01:25:32 *
```

## Expected Canvas Behavior
* The LCD widget prints the active date and time parsed from the RTC registers, updating every second.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `rtc.begin()` | Mounts the I2C interface of the DS3231 RTC module. |
| `configTime(...)` | Configures the timezone offsets for the internal ESP32 network time clock. |
| `rtc.adjust(...)` | Overwrites the DS3231 internal registers with the newly synchronized time values. |
| `rtc.now()` | Retrieves the current date and time from the DS3231 module registers. |

## Hardware & Safety Concept: NTP Sync and Battery Backup Fail-Safes
* **Battery Backup**: The DS3231 module features a backup coin cell battery (CR2032). When the main power (5V Vin) is cut, the battery keeps the internal oscillator running, preventing the clock from resetting.
* **NTP Drift Correction**: Mechanical crystals drift due to temperature fluctuations. DS3231 features a temperature-compensated crystal oscillator (TCXO), providing ±2 ppm accuracy (approx. 1 minute drift per year). Synchronizing with NTP on boot corrects any accumulated drift.

## Try This! (Challenges)
1. **OLED Status Display**: Upgrade the screen to an SSD1306 OLED (Project 60) and display a dynamic clock face.
2. **Local Sync Indicator**: Add a blue LED (GPIO 12) to light up when NTP time is synchronized.
3. **Buzzer Hourly Chime**: Sound a brief double beep on a buzzer (GPIO 15) at the start of every hour.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The clock resets on reboot | Backup battery dead | Verify that the CR2032 coin cell battery is installed on the DS3231 module |
| Time is offset by several hours | Timezone offset wrong | Verify that the `gmtOffset_sec` value corresponds to your local time zone |
| LCD does not print text | I2C address mismatch | Verify the I2C address in `LiquidCrystal_I2C` matches your LCD module (usually `0x27`) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [37 - ESP32 LCD Network Clock](../beginner/37-esp32-lcd-network-clock.md)
- [111 - ESP32 Rotary Encoder Web Menu Selector](111-esp32-rotary-encoder-web-menu-selector.md)
- [113 - ESP32 DS3231 RTC Alarm System IoT Control](113-esp32-ds3231-rtc-alarm-system-iot-control.md) (Next project)
