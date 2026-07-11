# 144 - RFID Access Logger

Scan RFID cards or fobs using an MFRC522 module, read the UID, fetch the current timestamp from a DS3231 real-time clock, and append a CSV record to an SD card for every successful scan.

## Goal

Learn how to chain three SPI/I2C peripherals — MFRC522 RFID reader (SPI), DS3231 RTC (I2C), and a microSD card module (SPI) — so that every badge tap produces a timestamped access-log entry that persists across power cycles.

## What You Will Build

An MFRC522 RFID reader is wired to the ARIES v3 SPI bus. A DS3231 RTC module provides the current date and time over I2C. Each time a card or fob is presented, the firmware reads the 4-byte UID, queries the RTC for a timestamp, formats a CSV line (`UID,YYYY-MM-DD,HH:MM:SS`), and appends it to `ACCESS.CSV` on the SD card. The Serial Monitor echoes every entry so you can monitor access events in real time.

## Parts Needed

| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MFRC522 RFID Module | `mfrc522` | Yes | Yes |
| DS3231 RTC Module | `ds3231` | Yes | Yes |
| MicroSD Card Module (SPI) | `sdcard` | Yes | Yes |
| MicroSD Card (FAT32) | — | No | Yes |
| RFID Card / Key Fob | — | No | Yes |
| Jumper Wires | — | No | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 | VCC | 3V3 | Red | MFRC522 operates at 3.3 V only |
| MFRC522 | GND | GND | Black | Common ground |
| MFRC522 | SCK | GPIO 6 (SPI0 SCK) | Yellow | SPI clock |
| MFRC522 | MOSI | GPIO 7 (SPI0 TX) | Blue | SPI data to module |
| MFRC522 | MISO | GPIO 4 (SPI0 RX) | Green | SPI data from module |
| MFRC522 | SDA (SS) | GPIO 5 | Orange | Chip-select for RFID module |
| MFRC522 | RST | GPIO 8 | Purple | Hardware reset line |
| SD Card Module | VCC | 3V3 | Red | Share rail with MFRC522 |
| SD Card Module | GND | GND | Black | Common ground |
| SD Card Module | SCK | GPIO 6 (SPI0 SCK) | Yellow | Shared SPI clock |
| SD Card Module | MOSI | GPIO 7 (SPI0 TX) | Blue | Shared MOSI |
| SD Card Module | MISO | GPIO 4 (SPI0 RX) | Green | Shared MISO |
| SD Card Module | CS | GPIO 9 | White | Chip-select for SD card |
| DS3231 RTC | VCC | 3V3 | Red | I2C power |
| DS3231 RTC | GND | GND | Black | Common ground |
| DS3231 RTC | SDA | GPIO 0 (I2C0 SDA) | Orange | I2C data |
| DS3231 RTC | SCL | GPIO 1 (I2C0 SCL) | Yellow | I2C clock |

> **Wiring tip:** Both the MFRC522 and the SD card share the same SPI0 bus (SCK, MOSI, MISO) but have separate chip-select (CS) lines — GPIO 5 for the RFID and GPIO 9 for the SD card. Never drive both CS lines LOW at the same time. The DS3231 sits on I2C0 at address 0x68 and does not interact with the SPI bus at all. Pre-format the SD card as FAT32 with a cluster size of 4 KB for best compatibility.

## Code

```cpp
// 144 - RFID Access Logger
// MFRC522 RFID + DS3231 RTC -> SD card CSV log
#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <RTClib.h>
#include <SD.h>

// Pin assignments
#define RFID_SS_PIN  5
#define RFID_RST_PIN 8
#define SD_CS_PIN    9

// Peripheral objects
MFRC522 rfid(RFID_SS_PIN, RFID_RST_PIN);
RTC_DS3231 rtc;

// State variables
int   sdReady    = 0;
int   rtcReady   = 0;
int   scanPending = 0;
float lastScanMs = 0;

// UID assembly variables (up to 4 bytes, no array declared)
int uidLen = 0;
int uidB0  = 0;
int uidB1  = 0;
int uidB2  = 0;
int uidB3  = 0;

// Timestamp variables
int tsYear = 0;
int tsMon  = 0;
int tsDay  = 0;
int tsHour = 0;
int tsMin  = 0;
int tsSec  = 0;

String uidString    = "";
String csvLine      = "";
String tsDateString = "";
String tsTimeString = "";

void setup() {
  Serial.begin(115200);
  Serial.println("=== RFID Access Logger ===");

  // Initialise SPI
  SPI.begin();

  // Initialise MFRC522
  rfid.PCD_Init();
  Serial.println("MFRC522 ready. Firmware version:");
  rfid.PCD_DumpVersionToSerial();

  // Initialise DS3231
  Wire.begin();
  if (!rtc.begin()) {
    Serial.println("ERROR: DS3231 RTC not found. Check I2C wiring.");
    rtcReady = 0;
  } else {
    rtcReady = 1;
    if (rtc.lostPower()) {
      Serial.println("RTC lost power - setting compile time.");
      rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));
    }
    Serial.println("DS3231 RTC ready.");
  }

  // Initialise SD card
  if (!SD.begin(SD_CS_PIN)) {
    Serial.println("ERROR: SD card initialisation failed. Check wiring and FAT32 format.");
    sdReady = 0;
  } else {
    sdReady = 1;
    Serial.println("SD card ready.");

    // Write CSV header if file does not already exist
    if (!SD.exists("ACCESS.CSV")) {
      File f = SD.open("ACCESS.CSV", FILE_WRITE);
      if (f) {
        f.println("UID,Date,Time");
        f.close();
        Serial.println("Created ACCESS.CSV with header.");
      }
    }
  }

  Serial.println("Waiting for RFID card...");
}

void loop() {
  // Only proceed if SD and RTC are both ready
  if (!sdReady || !rtcReady) {
    delay(1000);
    return;
  }

  // Poll for a new card (non-blocking check)
  if (!rfid.PICC_IsNewCardPresent()) {
    return;
  }
  if (!rfid.PICC_ReadCardSerial()) {
    return;
  }

  // Read UID bytes individually into named variables
  uidLen = rfid.uid.size;
  uidB0  = (uidLen > 0) ? rfid.uid.uidByte[0] : 0;
  uidB1  = (uidLen > 1) ? rfid.uid.uidByte[1] : 0;
  uidB2  = (uidLen > 2) ? rfid.uid.uidByte[2] : 0;
  uidB3  = (uidLen > 3) ? rfid.uid.uidByte[3] : 0;

  // Build UID hex string
  uidString = "";
  if (uidB0 < 0x10) uidString += "0";
  uidString += String(uidB0, HEX);
  uidString += ":";
  if (uidB1 < 0x10) uidString += "0";
  uidString += String(uidB1, HEX);
  uidString += ":";
  if (uidB2 < 0x10) uidString += "0";
  uidString += String(uidB2, HEX);
  uidString += ":";
  if (uidB3 < 0x10) uidString += "0";
  uidString += String(uidB3, HEX);

  // Fetch timestamp from RTC
  DateTime now = rtc.now();
  tsYear = now.year();
  tsMon  = now.month();
  tsDay  = now.day();
  tsHour = now.hour();
  tsMin  = now.minute();
  tsSec  = now.second();

  // Format date string YYYY-MM-DD
  tsDateString = String(tsYear) + "-";
  if (tsMon  < 10) tsDateString += "0";
  tsDateString += String(tsMon) + "-";
  if (tsDay  < 10) tsDateString += "0";
  tsDateString += String(tsDay);

  // Format time string HH:MM:SS
  tsTimeString = "";
  if (tsHour < 10) tsTimeString += "0";
  tsTimeString += String(tsHour) + ":";
  if (tsMin  < 10) tsTimeString += "0";
  tsTimeString += String(tsMin) + ":";
  if (tsSec  < 10) tsTimeString += "0";
  tsTimeString += String(tsSec);

  // Assemble CSV record
  csvLine = uidString + "," + tsDateString + "," + tsTimeString;

  // Append to SD card
  File logFile = SD.open("ACCESS.CSV", FILE_WRITE);
  if (logFile) {
    logFile.println(csvLine);
    logFile.close();
    Serial.print("LOGGED: ");
    Serial.println(csvLine);
  } else {
    Serial.println("ERROR: Could not open ACCESS.CSV for writing.");
  }

  // Halt the card so it does not register multiple scans
  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();

  // Brief debounce pause before next scan
  delay(1500);
}
```

## What to Click in MbedO

1. Drag **VEGA ARIES v3**, **MFRC522**, **DS3231 RTC**, and **SD Card** components onto the canvas.
2. Wire the MFRC522: **SCK → GPIO 6**, **MOSI → GPIO 7**, **MISO → GPIO 4**, **SS → GPIO 5**, **RST → GPIO 8**, **VCC → 3V3**, **GND → GND**.
3. Wire the SD Card module: **SCK → GPIO 6**, **MOSI → GPIO 7**, **MISO → GPIO 4**, **CS → GPIO 9**, **VCC → 3V3**, **GND → GND**.
4. Wire the DS3231: **SDA → GPIO 0**, **SCL → GPIO 1**, **VCC → 3V3**, **GND → GND**.
5. Paste the code into the editor and select **Interpreted Mode**.
6. Click **Run**.
7. In the MbedO canvas, click the **MFRC522 Scan** button to simulate a card tap. Observe the Serial Monitor for the logged CSV entry.

## Expected Output

Serial Monitor:
```
=== RFID Access Logger ===
MFRC522 ready. Firmware version:
DS3231 RTC ready.
SD card ready.
Created ACCESS.CSV with header.
Waiting for RFID card...
LOGGED: a3:2b:7f:c1,2026-07-11,10:35:22
LOGGED: a3:2b:7f:c1,2026-07-11,10:35:24
```

`ACCESS.CSV` on the SD card:
```
UID,Date,Time
a3:2b:7f:c1,2026-07-11,10:35:22
a3:2b:7f:c1,2026-07-11,10:35:24
```

## Expected Canvas Behavior

* Each simulated card tap in MbedO triggers one CSV line appended to `ACCESS.CSV`.
* The Serial Monitor echoes every log entry immediately after it is written.
* If the SD card is missing or the RTC is not detected, an error message is printed and the system pauses — it does not crash silently.

## Code Walkthrough

| Line | Check / Action |
| --- | --- |
| `#include <MFRC522.h>` | Imports the Miguel Balboa MFRC522 library for SPI RFID communication. |
| `#include <RTClib.h>` | Imports the Adafruit RTClib for DS3231 I2C real-time clock access. |
| `MFRC522 rfid(RFID_SS_PIN, RFID_RST_PIN)` | Creates the RFID driver bound to chip-select GPIO 5 and reset GPIO 8. |
| `rfid.PCD_Init()` | Resets and configures the MFRC522 over SPI. |
| `rfid.PICC_IsNewCardPresent()` | Non-blocking poll — returns true only when a card enters the RF field. |
| `rfid.uid.uidByte[n]` | Byte n of the scanned card's UID (up to 10 bytes; 4 bytes used here). |
| `rtc.now()` | Returns a `DateTime` struct holding year, month, day, hour, minute, second. |
| `SD.open("ACCESS.CSV", FILE_WRITE)` | Opens (or creates) the log file in append mode. |
| `rfid.PICC_HaltA()` | Puts the card into HALT state so the same card is not read again immediately. |
| `rfid.PCD_StopCrypto1()` | Clears the crypto state after authentication (good practice even without auth). |

## Hardware & Safety Concept

* **SPI Bus Sharing**: MFRC522 and the SD card module coexist on the same SPI0 bus. The firmware ensures only one CS line is driven LOW at any time — the MFRC522 library manages its own CS internally through the `MFRC522` constructor, and the `SD` library controls GPIO 9. Both peripherals are electrically isolated from each other when their CS is HIGH (idle).
* **Non-Volatile Logging**: Unlike RAM, the SD card retains the CSV file across power cycles. This is critical for security applications — even if the ARIES board loses power, every previously logged access event is preserved.
* **RTC Battery Backup**: The DS3231 has an onboard CR2032 coin-cell holder. As long as the battery is installed, the clock keeps running even when main power is removed, ensuring timestamps remain accurate for months without recalibration.

## Try This! (Challenges)

1. **Allow-list Check**: Add a known UID as a global `String allowedUID`. In `loop()`, after building `uidString`, compare it to `allowedUID` and print `"ACCESS GRANTED"` or `"ACCESS DENIED"` accordingly before writing the log.
2. **Scan Counter Column**: Add a global `int scanCount = 0` variable. Increment it on each successful scan and prefix each CSV line with the scan number (e.g., `1,a3:2b:7f:c1,2026-07-11,10:35:22`).

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| `ERROR: SD card initialisation failed` | CS wired to wrong pin or card not FAT32 | Confirm GPIO 9 is the SD CS pin; reformat the card as FAT32. |
| `ERROR: DS3231 RTC not found` | SDA/SCL swapped or missing pull-ups | Verify GPIO 0 = SDA, GPIO 1 = SCL; DS3231 modules include built-in pull-ups. |
| UID always reads `00:00:00:00` | MFRC522 not receiving power at 3.3 V | MFRC522 is NOT 5 V tolerant — ensure it is connected to 3V3, not VBUS. |
| Same card scanned multiple times instantly | Missing `PICC_HaltA()` call | Confirm the halt and StopCrypto1 lines are present at the bottom of `loop()`. |
| `ACCESS.CSV` not created | SD module MISO floating on idle | Add a 10 kΩ pull-up resistor between MISO and 3V3 to prevent bus contention. |

## Mode Notes

This project runs in MbedO **interpreted C++ mode**.

## Related Projects

- [143 - SD Card CSV Logger](143-sd-card-csv-logger.md)
- [145 - GPS Logger](145-gps-logger.md)
- [150 - Dual-protocol Telemetry Logger](150-dual-protocol-telemetry-logger.md)
