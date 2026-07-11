# 145 - ESP32 GPS Logger

Build a global positioning telemetry node that interfaces a NEO-6M GPS receiver over a hardware serial channel, parses NMEA coordinate strings using the `TinyGPS++` library, and outputs location data to the Serial Monitor.

## Goal
Learn how to read and decode standard NMEA-0183 sentences, extract coordinate values (latitude, longitude, altitude), and process satellite fix statuses.

## What You Will Build
A NEO-6M GPS module is connected to UART2 (RX2: GPIO 16, TX2: GPIO 17). The code reads raw serial data, feeds it to the `TinyGPS++` parser, and prints coordinates, altitude (meters), speed (km/h), and satellite counts to the Serial Monitor. It logs a "Waiting for Fix" state if no satellites are tracked.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| NEO-6M GPS Module | `gps_neo6m` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NEO-6M Module | TXD | GPIO16 (RX2) | Green | GPS Transmit line |
| NEO-6M Module | RXD | GPIO17 (TX2) | Yellow | GPS Receive line |
| NEO-6M Module | VCC / GND | 3V3 / GND | Red / Black | Power rails |

> **Wiring tip:** Standard NEO-6M GPS modules run at a default baud rate of **9600**. Make sure to connect the module's TXD pin to the ESP32's RX2 (GPIO 16) and the RXD pin to TX2 (GPIO 17).

## Code
```cpp
// GPS Logger (NEO-6M GPS Parser)
#include <TinyGPS++.h>

#define RX2_PIN 16
#define TX2_PIN 17

TinyGPSPlus gps;

void setup() {
  Serial.begin(115200);
  
  // Initialize UART2 for GPS at 9600 baud
  Serial2.begin(9600, SERIAL_8N1, RX2_PIN, TX2_PIN);
  
  Serial.println("NEO-6M GPS Logger starting...");
  Serial.println("Note: GPS requires outdoor line-of-sight to acquire satellites.");
}

void loop() {
  // Feed raw serial data to the TinyGPS++ parser
  while (Serial2.available() > 0) {
    gps.encode(Serial2.read());
  }
  
  // Every 2 seconds, print current GPS coordinates
  static unsigned long lastPrint = 0;
  if (millis() - lastPrint > 2000) {
    lastPrint = millis();
    
    Serial.print("Satellites: ");
    Serial.println(gps.satellites.value());
    
    if (gps.location.isValid()) {
      Serial.print("Latitude:  "); Serial.println(gps.location.lat(), 6);
      Serial.print("Longitude: "); Serial.println(gps.location.lng(), 6);
      Serial.print("Altitude:  "); Serial.print(gps.altitude.meters(), 1); Serial.println(" m");
      Serial.print("Speed:     "); Serial.print(gps.speed.kmph(), 1); Serial.println(" km/h");
      
      // Date & Time
      Serial.print("Date:      ");
      Serial.printf("%02d/%02d/%04d\n", gps.date.day(), gps.date.month(), gps.date.year());
      Serial.print("Time:      ");
      Serial.printf("%02d:%02d:%02d UTC\n", gps.time.hour(), gps.time.minute(), gps.time.second());
      
      Serial.println("----------------------------------------");
    } else {
      Serial.println(">> Waiting for GPS Fix (Move outdoors) <<");
      Serial.println("----------------------------------------");
    }
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **NEO-6M GPS** onto the canvas.
2. Wire GPS TXD to **GPIO16** and RXD to **GPIO17**.
3. Paste the code and click **Run**.
4. Adjust the GPS coordinate sliders on the simulated widget. Watch the decoded values update in the Serial Monitor.

## Expected Output
Serial Monitor:
```
NEO-6M GPS Logger starting...
Satellites: 6
Latitude:  12.971598
Longitude: 77.594562
Altitude:  920.5 m
Speed:     0.0 km/h
Date:      11/07/2026
Time:      00:54:12 UTC
----------------------------------------
```

## Expected Canvas Behavior
* The GPS widget sends mock serial strings over the UART2 bus.
* Adjusting the coordinates or satellite sliders on the GPS widget updates the decoded information in the Serial Monitor.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `gps.encode(Serial2.read())` | Parses incoming bytes and extracts standard NMEA sentences. |
| `gps.location.isValid()` | Returns true if the GPS module has acquired a valid satellite lock and calculated coordinates. |
| `gps.location.lat()` | Returns the calculated latitude coordinate in decimal degrees. |

## Hardware & Safety Concept: GPS Satellite Triangulation and Antenna Placement
GPS receivers require a clear line-of-sight view of the sky to lock onto radio signals from GPS satellites orbiting 20,000 km above. A receiver needs signals from at least **four satellites** (triangulation) to compute latitude, longitude, altitude, and clock time. Indoors or in dense urban areas, signals are blocked, preventing a location fix.

## Try This! (Challenges)
1. **Fix indicator LED**: Add an LED on GPIO 12 that stays ON if a valid GPS location fix is acquired, and blinks if searching.
2. **Interactive OLED Map coordinates**: Add an OLED display (Project 60) showing Lat/Lon and current UTC time.
3. **Speed Trap Buzzer Alarm**: Sound a buzzer if the GPS velocity exceeds a speed limit (e.g. 5.0 km/h).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Latitude shows 0.0 or "Waiting for Fix" | Indoor placement (no satellite signal) | Move the antenna near a window or outdoors |
| Serial Monitor shows no data | RXD and TXD lines swapped | Swap the connections between the GPS module and GPIO 16/17 |
| Output shows corrupted characters | Baud rate mismatch | Verify both the GPS module and `Serial2.begin` are configured to 9600 baud |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [89 - ESP32 HC-05 Bluetooth Serial UART Link](../intermediate/89-esp32-hc05-bluetooth-serial-uart-link.md)
- [146 - ESP32 GPS Location Display](146-esp32-gps-location-display.md) (Next project)
- [147 - ESP32 GPS Velocity Logger](147-esp32-gps-velocity-logger.md)
