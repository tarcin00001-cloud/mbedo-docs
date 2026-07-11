# 35 - ESP32 NTP Time Zone Offset Adjustments

Configure the ESP32 to calculate local time offsets relative to Coordinated Universal Time (UTC) using GMT time zone offsets and Daylight Saving Time (DST) parameters, and print local date-time strings to the Serial Monitor.

## Goal
Learn how to configure GMT offsets, handle Daylight Saving Time (DST) offsets in seconds, adjust local time zones, and format timestamps.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It synchronizes its clock with the NTP server `pool.ntp.org`. By adjusting timezone parameters (GMT offset and DST offset), it calculates and displays the local time in Indian Standard Time (IST: UTC+5:30) and Eastern Standard Time (EST: UTC-5) side by side.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// NTP Time Zone offset adjustments (Local timezone calculation)
#include <WiFi.h>
#include "time.h"

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const char* ntpServer = "pool.ntp.org";

// Indian Standard Time (IST) Configuration
// IST is UTC+5:30 -> (5 hours * 3600) + (30 minutes * 60) = 19800 seconds
const long  gmtOffset_IST = 19800;      
const int   daylightOffset_IST = 0;     // No DST in India

// Eastern Standard Time (EST) / Eastern Daylight Time (EDT) Configuration
// EST is UTC-5 -> -5 * 3600 = -18000 seconds
const long  gmtOffset_EST = -18000;     
const int   daylightOffset_EST = 3600;  // DST active (add 1 hour = 3600 seconds)

void printTimeForZone(const char* zoneLabel, long gmtOffset, int dstOffset) {
  struct tm timeinfo;
  
  // 1. Re-configure the time provider with target zone parameters
  configTime(gmtOffset, dstOffset, ntpServer);
  
  // 2. Read time from RTC
  if (!getLocalTime(&timeinfo)) {
    Serial.printf("[%s] Failed to retrieve time!\n", zoneLabel);
    return;
  }
  
  // 3. Print formatted date-time string
  char timeString[64];
  strftime(timeString, sizeof(timeString), "%A, %B %d %Y %H:%M:%S", &timeinfo);
  
  Serial.printf("%-10s Time: %s (Offset: %lds, DST: %ds)\n", 
                zoneLabel, 
                timeString, 
                gmtOffset, 
                dstOffset);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 Timezone Offset Adjuster");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Seed the initial NTP sync
  configTime(0, 0, ntpServer);
  struct tm timeinfo;
  while (!getLocalTime(&timeinfo)) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nNTP Synced.");
  Serial.println("==================================\n");
}

void loop() {
  // Read and print time for different configured zones
  printTimeForZone("IST (India)", gmtOffset_IST, daylightOffset_IST);
  printTimeForZone("EST (USA)",   gmtOffset_EST, daylightOffset_EST);
  Serial.println("------------------------------------------------------------");
  
  delay(5000); // Check every 5 seconds
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Watch the local times print for both zones.

## Expected Output
Serial Monitor:
```
==================================
ESP32 Timezone Offset Adjuster
==================================
WiFi Connected.
NTP Synced.
==================================

IST (India) Time: Saturday, July 11 2026 06:50:00 (Offset: 19800s, DST: 0s)
EST (USA)   Time: Friday, July 10 2026 21:20:00 (Offset: -18000s, DST: 3600s)
------------------------------------------------------------
```

## Expected Canvas Behavior
* The Serial Monitor logs the synchronized dates and times for both target zones (IST and EST) every 5 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `gmtOffset_IST = 19800` | Calculates the time offset in seconds for Indian Standard Time (5.5 hours). |
| `gmtOffset_EST = -18000` | Calculates the time offset in seconds for Eastern Standard Time (-5 hours). |
| `configTime(...)` | Re-configures the timezone settings, updating the RTC calculation. |
| `strftime(...)` | Formats date-time structures into human-readable strings. |

## Hardware & Safety Concept: Epoch Time and Time Zone Math
Computers and networks keep time using **UTC Epoch Time** (the number of seconds elapsed since January 1, 1970). To display local time, the controller must calculate the offset from UTC:
1. **GMT Offset**: Shifts the clock based on the time zone relative to the Prime Meridian.
2. **Daylight Saving Time (DST) Offset**: Shifts the clock by 1 hour (3600 seconds) during summer months.
Enforcing these offsets ensures the clock remains accurate.

## Try This! (Challenges)
1. **OLED Dual Clock HUD**: Add an OLED screen (Project 60) and display both time zones side by side.
2. **Dynamic Zone Selector**: Add a button on GPIO 12 to cycle through a list of pre-configured time zones.
3. **Automatic DST Rule**: Write code to automatically enable or disable the DST offset based on the calendar month.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Date always displays 1970 | NTP sync failed | Verify that the ESP32 is connected to the network before initializing NTP |
| Time is offset by 1 hour | DST setting incorrect | Verify that the DST offset parameter matches the local season (active/inactive) |
| System logs display garbage characters | Baud rate mismatch | Ensure the Serial Monitor baud rate matches the sketch setting (115200) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [34 - ESP32 NTP Time Server sync](34-esp32-ntp-time-server-sync.md)
- [36 - ESP32 OLED Digital Clock (NTP synced)](36-esp32-oled-digital-clock-ntp-synced.md)
- [37 - ESP32 LCD Network Clock](37-esp32-lcd-network-clock.md)
