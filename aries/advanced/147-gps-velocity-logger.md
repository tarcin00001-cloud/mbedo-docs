# 147 - GPS Velocity Logger

Parse ground speed from a NEO-6M GPS module's `$GPVTG` sentence and log each reading as a timestamped CSV row to a microSD card via SPI.

## Goal

Learn how to extract the speed-over-ground (SOG) field from the `$GPVTG` NMEA sentence, maintain a session log on an SD card that accumulates speed samples at 1 Hz, and use MbedO's interpreted mode state-machine pattern to handle NMEA accumulation, field extraction, and SD card writes — all without arrays, helper functions, or loops inside `loop()`.

## What You Will Build

A NEO-6M GPS module streams NMEA data to ARIES `Serial2` (GPIO 16/17). The firmware identifies `$GPVTG` sentences, extracts speed in km/h, and appends a CSV row (`SampleNo,SpeedKmh,SpeedKnots`) to `VEL.CSV` on the SD card. The Serial Monitor echoes each logged entry. A session sample counter increments with every write.

## Parts Needed

| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| u-blox NEO-6M GPS Module | `neo6m` | Yes | Yes |
| MicroSD Card Module (SPI) | `sdcard` | Yes | Yes |
| MicroSD Card (FAT32) | — | No | Yes |
| Active GPS Antenna (SMA) | — | No | Yes |
| Jumper Wires | — | No | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NEO-6M | VCC | 3V3 | Red | 3.3 V GPS power |
| NEO-6M | GND | GND | Black | Common ground |
| NEO-6M | TX | GPIO 17 (UART2 RX) | Green | GPS data → ARIES |
| NEO-6M | RX | GPIO 16 (UART2 TX) | Blue | ARIES → GPS (optional config) |
| SD Card Module | VCC | 3V3 | Red | Shared 3V3 rail |
| SD Card Module | GND | GND | Black | Common ground |
| SD Card Module | SCK | GPIO 6 (SPI0 SCK) | Yellow | SPI clock |
| SD Card Module | MOSI | GPIO 7 (SPI0 TX) | Blue | SPI data to SD |
| SD Card Module | MISO | GPIO 4 (SPI0 RX) | Green | SPI data from SD |
| SD Card Module | CS | GPIO 9 | White | Chip-select for SD |

> **Wiring tip:** The NEO-6M UART lines (GPIO 16/17) and the SPI SD card lines (GPIO 4/6/7/9) are independent buses — they do not share signals. Pre-format the SD card as FAT32. Ensure the SD module's MISO line is pulled up to 3V3 via a 10 kΩ resistor if you observe initialisation failures.

## Code

```cpp
// 147 - GPS Velocity Logger
// NEO-6M $GPVTG speed -> SD Card CSV log
#include <SPI.h>
#include <SD.h>

#define SD_CS_PIN   9
#define GPS_BAUD    9600
#define SERIAL_BAUD 115200

// SD state
int sdReady = 0;

// NMEA sentence accumulator
String nmeaSentence  = "";
int    sentenceReady = 0;

// Parsed speed values
float speedKmh   = 0.0;
float speedKnots = 0.0;
int   gpsFixMode = 0;  // field 9 of GPVTG: 'A'=autonomous, 'N'=not valid

// Sample counter
int sampleNo = 0;

// Field extraction state variables
int    fStart = 0;
int    fEnd   = 0;
String fVal   = "";

// CSV line
String csvLine = "";

// Incoming character
char gpsCh = 0;

// SD write throttle
unsigned long lastLogMs = 0;
unsigned long nowMs     = 0;

void setup() {
  Serial.begin(SERIAL_BAUD);
  Serial2.begin(GPS_BAUD);
  SPI.begin();

  Serial.println("=== GPS Velocity Logger ===");

  if (!SD.begin(SD_CS_PIN)) {
    Serial.println("ERROR: SD card initialisation failed.");
    sdReady = 0;
  } else {
    sdReady = 1;
    Serial.println("SD card ready.");

    if (!SD.exists("VEL.CSV")) {
      File f = SD.open("VEL.CSV", FILE_WRITE);
      if (f) {
        f.println("SampleNo,SpeedKmh,SpeedKnots");
        f.close();
        Serial.println("Created VEL.CSV with header.");
      }
    }
  }

  Serial.println("Waiting for $GPVTG sentence with valid fix...");
}

void loop() {
  // ---- Step 1: Accumulate one complete NMEA sentence ----
  sentenceReady = 0;

  if (Serial2.available() > 0) {
    gpsCh = (char)Serial2.read();

    if (gpsCh == '$') {
      nmeaSentence = "$";
    } else if (gpsCh == '\n') {
      sentenceReady = 1;
    } else if (gpsCh != '\r') {
      nmeaSentence += gpsCh;
    }
  }

  // ---- Step 2: Parse $GPVTG sentences only ----
  if (sentenceReady && nmeaSentence.startsWith("$GPVTG")) {
    // GPVTG sentence field layout (comma index 0-based):
    // 0=$GPVTG, 1=TrackTrue, 2=T, 3=TrackMag, 4=M,
    // 5=SpeedKnots, 6=N, 7=SpeedKmh, 8=K, 9=ModeIndicator

    // Field 5: speed in knots
    fStart = nmeaSentence.indexOf(',', 0);                    // after field 0
    fStart = nmeaSentence.indexOf(',', fStart + 1);           // after field 1
    fStart = nmeaSentence.indexOf(',', fStart + 1);           // after field 2
    fStart = nmeaSentence.indexOf(',', fStart + 1);           // after field 3
    fStart = nmeaSentence.indexOf(',', fStart + 1);           // after field 4
    fEnd   = nmeaSentence.indexOf(',', fStart + 1);           // end of field 5
    fVal   = nmeaSentence.substring(fStart + 1, fEnd);
    speedKnots = fVal.toFloat();

    // Field 7: speed in km/h (skip 2 more commas: field 6=N, then 7=value)
    fStart = fEnd;                                            // was end of field 5
    fStart = nmeaSentence.indexOf(',', fStart + 1);           // after field 6 (N)
    fEnd   = nmeaSentence.indexOf(',', fStart + 1);           // end of field 7
    fVal   = nmeaSentence.substring(fStart + 1, fEnd);
    speedKmh = fVal.toFloat();

    // Field 9: mode indicator (A=autonomous, N=not valid)
    fStart = fEnd;
    fStart = nmeaSentence.indexOf(',', fStart + 1);           // after field 8 (K)
    int fEnd9 = nmeaSentence.indexOf('*', fStart + 1);        // end at checksum *
    if (fEnd9 == -1) fEnd9 = nmeaSentence.length();
    fVal = nmeaSentence.substring(fStart + 1, fEnd9);
    // Mode 'A' = valid autonomous GPS fix
    gpsFixMode = (fVal == "A") ? 1 : 0;
  }

  // ---- Step 3: Log to SD card once per second if fix is valid ----
  nowMs = millis();
  if (!sdReady) return;
  if (!gpsFixMode) return;
  if (nowMs - lastLogMs < 1000) return;

  lastLogMs = nowMs;
  sampleNo++;

  csvLine = String(sampleNo) + "," + String(speedKmh, 2) + "," + String(speedKnots, 2);

  File logFile = SD.open("VEL.CSV", FILE_WRITE);
  if (logFile) {
    logFile.println(csvLine);
    logFile.close();
    Serial.print("LOGGED #");
    Serial.print(sampleNo);
    Serial.print("  Speed: ");
    Serial.print(speedKmh, 2);
    Serial.print(" km/h  (");
    Serial.print(speedKnots, 2);
    Serial.println(" kn)");
  } else {
    Serial.println("ERROR: Could not open VEL.CSV.");
  }
}
```

## What to Click in MbedO

1. Drag **VEGA ARIES v3**, **NEO-6M GPS**, and **SD Card** components onto the canvas.
2. Wire the NEO-6M: **TX → GPIO 17**, **RX → GPIO 16**, **VCC → 3V3**, **GND → GND**.
3. Wire the SD Card: **SCK → GPIO 6**, **MOSI → GPIO 7**, **MISO → GPIO 4**, **CS → GPIO 9**, **VCC → 3V3**, **GND → GND**.
4. Paste the code into the editor and select **Interpreted Mode**.
5. Click **Run**.
6. Use the NEO-6M widget's **Speed** slider to set simulated ground speed (in km/h). The firmware logs a new sample each second.
7. Open `VEL.CSV` from the SD card widget file browser to inspect recorded data.

## Expected Output

Serial Monitor:
```
=== GPS Velocity Logger ===
SD card ready.
Created VEL.CSV with header.
Waiting for $GPVTG sentence with valid fix...
LOGGED #1  Speed: 45.37 km/h  (24.49 kn)
LOGGED #2  Speed: 45.37 km/h  (24.49 kn)
LOGGED #3  Speed: 62.10 km/h  (33.54 kn)
```

`VEL.CSV` on SD card:
```
SampleNo,SpeedKmh,SpeedKnots
1,45.37,24.49
2,45.37,24.49
3,62.10,33.54
```

## Expected Canvas Behavior

* No entries are logged until the GPS reports a valid fix (`mode = A` in `$GPVTG`).
* Speed samples are appended to `VEL.CSV` once per second.
* Adjusting the MbedO speed slider causes the next logged sample to reflect the new value.
* If the SD card fails, the Serial Monitor prints an error and the logger halts gracefully without crashing.

## Code Walkthrough

| Line | Check / Action |
| --- | --- |
| `nmeaSentence.startsWith("$GPVTG")` | Selects only Track Made Good and Speed Over Ground sentences. |
| Field 5 extraction | Walks through comma positions using repeated `indexOf(',', startPos)` calls — no split or array needed. |
| `speedKnots = fVal.toFloat()` | Parses the knots field directly from the substring. |
| Field 7 extraction | Skips one more comma past the `N` unit indicator to reach the km/h value. |
| `fVal == "A"` | Mode indicator `A` = autonomous valid fix; `N` = data not valid. Only log when `A`. |
| `nowMs - lastLogMs < 1000` | Enforces a 1-second interval between SD writes without using `delay()`. |
| `sampleNo++` | Monotonically increasing session sample counter written to CSV column 1. |
| `SD.open("VEL.CSV", FILE_WRITE)` | Opens the file in append mode; creates it if it does not exist. |

## Hardware & Safety Concept

* **$GPVTG Sentence**: NMEA Track Made Good and Ground Speed sentence provides speed in both knots and km/h, plus a mode indicator that distinguishes a valid autonomous fix from estimated (dead-reckoning) or invalid data. Unlike `$GPRMC` which also carries speed, `$GPVTG` is dedicated to velocity and easier to parse for this purpose.
* **SD Write Frequency**: Opening and closing the file on every write (rather than keeping it open) ensures each record is flushed to the FAT filesystem. If power is lost mid-session, all previously closed records are intact. Writing at 1 Hz produces approximately 35 bytes per second — negligible for a FAT32 card but important to track for multi-hour deployments.
* **Non-Blocking Architecture**: The `nowMs - lastLogMs` timer pattern prevents SD write overhead from blocking the UART receive path. If `loop()` were blocked for even 50 ms, incoming NMEA characters could overflow the hardware UART FIFO, causing sentence corruption. Keeping writes short and infrequent protects data integrity.

## Try This! (Challenges)

1. **Maximum Speed Tracker**: Add a global `float maxSpeedKmh = 0.0` variable. After parsing `speedKmh`, compare it and update `maxSpeedKmh` if the new reading is larger. Log the session maximum as an extra column in each CSV row.
2. **Distance Estimation**: Add a global `float totalDistKm = 0.0` variable. After each successful 1-second log, add `speedKmh / 3600.0` to `totalDistKm` (distance = speed × time). Log this as a fourth column to track cumulative distance travelled.

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| `ERROR: SD card initialisation failed` | CS pin wrong or card not FAT32 | Confirm CS is GPIO 9; reformat card as FAT32 on a PC. |
| No entries logged; Serial shows no LOGGED lines | GPS fix mode is `N` (no valid fix) | Ensure GPS has a clear sky view or use MbedO simulator speed slider to inject a valid $GPVTG sentence. |
| Speed always reads 0.00 | $GPVTG field index off by one | Verify the comma-walking logic in code; count commas manually in a raw NMEA sentence. |
| `VEL.CSV` missing after power cycle | File not closed before power loss | The code closes after each write; ensure no hard power-cut during an active `SD.open()` call. |

## Mode Notes

This project runs in MbedO **interpreted C++ mode**.

## Related Projects

- [145 - GPS Logger](145-gps-logger.md)
- [146 - GPS Location Display](146-gps-location-display.md)
- [150 - Dual-protocol Telemetry Logger](150-dual-protocol-telemetry-logger.md)
