# 147 - ESP32 GPS Velocity Logger

Build a vehicle tracking logger that parses speed and location from a NEO-6M GPS module and records them as CSV entries to an SPI SD card, using green/red LEDs to indicate write success or fix status.

## Goal
Learn how to run GPS parser libraries and SD card file writing concurrently on shared SPI buses, logging geographic velocity data.

## What You Will Build
A NEO-6M GPS module is connected to UART2. An SD card reader is connected via SPI (CS: 15). A Green LED (write confirmation) is on GPIO 12, and a Red LED (no GPS lock) on GPIO 13. Every 5 seconds, if a valid GPS fix exists, the speed (km/h) and coordinates are appended to `/gps_log.csv` on the SD card.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| NEO-6M GPS Module | `gps_neo6m` | Yes | Yes |
| MicroSD Card Shield (SPI) | `sd_card` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 330 Ω Resistors (2) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NEO-6M Module | TXD / RXD | GPIO16 / GPIO17 | Green / Yellow | GPS Serial lines |
| SD Card Module | CS / SCK / MISO / MOSI | GPIO15 / 18 / 19 / 23 | Purple/Yellow/Green/Blue | SPI interface |
| Green LED | Anode (+) | GPIO12 via 330 Ω | Green | Write confirmation |
| Red LED | Anode (+) | GPIO13 via 330 Ω | Red | GPS search warning |
| Common ground | GND / Cathodes | GND | Black | Shared reference |

> **Wiring tip:** The SD CS line connects to GPIO 15. Standard hardware serial UART2 pins (GPIO 16 and 17) handle the GPS. Power the SD card and GPS from the appropriate rails.

## Code
```cpp
// GPS Velocity Logger (Speed -> SD Card)
#include <SPI.h>
#include <FS.h>
#include <SD.h>
#include <TinyGPS++.h>

#define RX2_PIN 16
#define TX2_PIN 17
const int SD_CS = 15;

const int LED_GREEN = 12;
const int LED_RED = 13;

TinyGPSPlus gps;

unsigned long lastLogTime = 0;
const unsigned long LOG_INTERVAL_MS = 5000; // Log every 5 seconds

void writeCSVHeader(fs::FS &fs, const char * path) {
  if (!fs.exists(path)) {
    Serial.println("Creating GPS CSV log file...");
    File file = fs.open(path, FILE_WRITE);
    if (file) {
      file.println("Time(ms),Lat,Lon,Speed(kmph)");
      file.close();
    }
  }
}

void logGPS(fs::FS &fs, const char * path, float lat, float lon, float speed) {
  File file = fs.open(path, FILE_APPEND);
  if (!file) {
    Serial.println("Failed to open log file!");
    return;
  }
  
  String csvRow = String(millis()) + "," + String(lat, 6) + "," + String(lon, 6) + "," + String(speed, 1);
  if (file.println(csvRow)) {
    Serial.print("Logged: "); Serial.println(csvRow);
    digitalWrite(LED_GREEN, HIGH);
    delay(100);
    digitalWrite(LED_GREEN, LOW);
  } else {
    Serial.println("Write failed!");
  }
  file.close();
}

void setup() {
  Serial.begin(115200);
  Serial2.begin(9600, SERIAL_8N1, RX2_PIN, TX2_PIN);
  
  pinMode(LED_GREEN, OUTPUT);
  pinMode(LED_RED, OUTPUT);
  digitalWrite(LED_GREEN, LOW);
  digitalWrite(LED_RED, LOW);
  
  Serial.println("Mounting SD Card...");
  if (!SD.begin(SD_CS)) {
    Serial.println("SD Card Mount Failed!");
    while(1) {
      digitalWrite(LED_RED, HIGH); delay(200); digitalWrite(LED_RED, LOW); delay(200);
    }
  }
  
  writeCSVHeader(SD, "/gps_log.csv");
  Serial.println("GPS Velocity Logger active. Waiting for satellite lock.");
}

void loop() {
  // Feed GPS parser
  while (Serial2.available() > 0) {
    gps.encode(Serial2.read());
  }
  
  unsigned long now = millis();
  
  if (now - lastLogTime >= LOG_INTERVAL_MS) {
    if (gps.location.isValid()) {
      digitalWrite(LED_RED, LOW); // Turn off search warning
      
      float lat = gps.location.lat();
      float lon = gps.location.lng();
      float speedKmph = gps.speed.kmph();
      
      logGPS(SD, "/gps_log.csv", lat, lon, speedKmph);
    } else {
      Serial.println("No GPS fix. Skipping log.");
      // Flash Red LED to show searching state
      digitalWrite(LED_RED, HIGH);
      delay(100);
      digitalWrite(LED_RED, LOW);
    }
    
    lastLogTime = now;
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **NEO-6M GPS**, **SD Card**, and two **LEDs** onto the canvas.
2. Wire GPS to **GPIO16/GPIO17**, SD CS to **GPIO15**, Green LED to **GPIO12**, and Red LED to **GPIO13**.
3. Paste the code and click **Run**.
4. Adjust GPS satellite count. Watch the Red LED blink if there is no fix.
5. Set satellites > 4 and change the speed slider. Watch the Green LED flash every 5 seconds as it logs to `/gps_log.csv`.

## Expected Output
Serial Monitor:
```
Mounting SD Card...
GPS Velocity Logger active.
No GPS fix. Skipping log.
Logged: 10000,12.971598,77.594562,35.4
Logged: 15000,12.971620,77.594580,42.1
```

CSV file contents (`/gps_log.csv`):
```
Time(ms),Lat,Lon,Speed(kmph)
10000,12.971598,77.594562,35.4
15000,12.971620,77.594580,42.1
```

## Expected Canvas Behavior
* At startup, the SD card mounts.
* If the GPS coordinates are invalid, the Red LED blinks every 5 seconds.
* Once the GPS coordinates are valid, the Green LED flashes every 5 seconds, writing velocity data to `/gps_log.csv`.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `gps.location.isValid()` | Verifies that a valid coordinate set has been parsed. |
| `gps.speed.kmph()` | Retrieves the vehicle speed in kilometers per hour. |
| `String(lat, 6)` | Formats double-precision coordinates with 6 decimal places. |

## Hardware & Safety Concept: Vehicle Telemetry Auditing and SD Card Flushing
Vehicle telemetry loggers record trip details. Speed calculations are derived from the Doppler shift of GPS carrier waves, making it highly accurate when moving. If the SD card is pulled out while writing, the file system will corrupt. Status LEDs provide drivers visual feedback that logging is active and safe to power down only when writing has ceased.

## Try This! (Challenges)
1. **Odometer Integration**: Integrate the distance traveled by calculating the distance between successive coordinates (Haversine formula) and log it.
2. **Speeding Warning**: Sound a buzzer (GPIO 4) if the GPS speed exceeds a target speed limit (e.g. 50 km/h).
3. **Log status screen**: Add an OLED HUD (Project 60) showing files logged and coordinates in real time.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Velocity is always 0.0 | GPS is static | Walk outside or slide the speed slider on the virtual GPS widget |
| Red LED flashes constantly | GPS antenna shielded | Ensure outdoor line of sight; increase satellite count in the simulator |
| SD card fails to mount | Pin mapping conflict | Verify SD CS pin is wired to GPIO 15, not GPIO 5 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [145 - ESP32 GPS Logger](145-esp32-gps-logger.md)
- [146 - ESP32 GPS Location Display](146-esp32-gps-location-display.md)
- [137 - ESP32 SD Card Temperature Logger](137-esp32-sd-card-temperature-logger.md)
