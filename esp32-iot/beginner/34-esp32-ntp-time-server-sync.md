# 34 - ESP32 NTP Time Server Sync

Configure the ESP32 to establish network synchronization with atomic time servers over the Network Time Protocol (NTP), configure GMT offsets, and format and print local timestamps to the Serial Monitor.

## Goal
Learn how to configure the internal ESP32 Real-Time Clock (RTC), query NTP servers, convert epoch timestamps, and format date-time strings.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It synchronizes its internal clock with the network NTP server `pool.ntp.org`. Once synced, it prints the current UTC date and time to the Serial Monitor every second.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// NTP Time Server sync (Sync local RTC with network time)
#include <WiFi.h>
#include "time.h"

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// NTP Configuration
const char* ntpServer = "pool.ntp.org";
const long  gmtOffset_sec = 0;      // UTC time (no offset)
const int   daylightOffset_sec = 0; // No daylight savings offset

void printLocalTime() {
  struct tm timeinfo;
  
  // 1. Retrieve the local time from the ESP32 RTC
  // Returns false if the clock has not been initialized
  if (!getLocalTime(&timeinfo)) {
    Serial.println("Failed to obtain time from RTC!");
    return;
  }
  
  // 2. Print formatted time string to Serial Monitor
  // Format details: %A=Day, %B=Month, %d=Date, %Y=Year, %H:%M:%S=Time
  Serial.print("Current Time: ");
  Serial.println(&timeinfo, "%A, %B %d %Y %H:%M:%S");
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 NTP Clock Sync Starting");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // 3. Initialize NTP synchronization
  // Configures the local time provider and starts background sync
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  
  Serial.println("Waiting for NTP server response...");
  
  // Wait for the clock to sync
  struct tm timeinfo;
  int retry = 0;
  while (!getLocalTime(&timeinfo) && retry < 20) {
    delay(500);
    Serial.print(".");
    retry++;
  }
  
  if (retry < 20) {
    Serial.println("\nNTP Synchronization successful.");
  } else {
    Serial.println("\nNTP Synchronization failed! Using un-synchronized RTC.");
  }
  Serial.println("==================================\n");
}

void loop() {
  // Read and print local time once every second
  printLocalTime();
  
  delay(1000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Watch the synchronized UTC time print in real time.

## Expected Output
Serial Monitor:
```
==================================
ESP32 NTP Clock Sync Starting
==================================
WiFi Connected.
Waiting for NTP server response...
....
NTP Synchronization successful.
==================================

Current Time: Saturday, July 11 2026 01:20:00
Current Time: Saturday, July 11 2026 01:20:01
```

## Expected Canvas Behavior
* The Serial Monitor prints the updated date-time string every second.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `configTime(...)` | Configures the local time zone offset and triggers background NTP synchronization. |
| `getLocalTime(&timeinfo)` | Populates the `tm` structure with the date and time from the internal RTC. |
| `%A, %B %d %Y ...` | Standard C format specifiers to compile the date-time string. |

## Hardware & Safety Concept: Network Time Protocol (NTP) and the RTC
Microcontrollers do not have an accurate internal clock and lose time quickly when powered down. The **NTP (Network Time Protocol)** syncs the local clock with atomic time servers over the network using UDP port 123. The ESP32's built-in **Real-Time Clock (RTC)** maintains the time in the background once synchronized, allowing scheduling tasks without manual adjustments.

## Try This! (Challenges)
1. **OLED Clock Display**: Add an OLED screen (Project 60) and display a digital clock.
2. **Dynamic Alarm**: Trigger a buzzer beep (GPIO 15) when the second counter reaches zero (on the minute).
3. **Periodic Sync Scheduler**: Re-trigger NTP synchronization every 24 hours to correct for internal clock drift.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Time prints as 1970 (epoch start) | NTP sync failed | Verify that the ESP32 is connected to the internet before initializing NTP |
| Time is slightly offset | Clock drift | Re-trigger `configTime` periodically to sync the clock with NTP servers |
| Time zone offset is incorrect | UTC offset parameter wrong | Adjust the `gmtOffset_sec` parameter in the code to match your local time zone |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [35 - ESP32 NTP Time Zone offset adjustments](35-esp32-ntp-time-zone-offset-adjustments.md)
- [36 - ESP32 OLED Digital Clock (NTP synced)](36-esp32-oled-digital-clock-ntp-synced.md)
- [37 - ESP32 LCD Network Clock](37-esp32-lcd-network-clock.md)
