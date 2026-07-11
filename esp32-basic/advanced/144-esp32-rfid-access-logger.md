# 144 - ESP32 RFID Access Logger

Build a security access controller that reads RFID cards using an MFRC522 scanner, matches card UIDs against a master list to grant/deny access (lighting green/red status LEDs), and logs every event to an SPI SD card CSV file.

## Goal
Learn how to run multiple SPI devices (RFID and SD Card) concurrently by managing separate Chip Select (CS) lines, and format audit logs.

## What You Will Build
An MFRC522 RFID scanner and an SD card reader share the SPI bus. CS for RFID is on GPIO 5, CS for SD is on GPIO 15. A Green LED is on GPIO 12, and a Red LED on GPIO 13. When a card is scanned, its UID is compared to a master key. Granted or denied events are logged as CSV entries `/access.csv` on the SD card.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MFRC522 RFID Reader | `rfid` | Yes | Yes |
| MicroSD Card Shield (SPI) | `sd_card` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 330 Ω Resistors (2) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| SPI Bus (shared) | SCK / MISO / MOSI | GPIO18 / 19 / 23 | Yellow/Green/Blue | Shared SPI lines |
| MFRC522 Module | SDA (CS) / RST | GPIO5 / GPIO4 | Orange / Red | RFID controls |
| SD Card Module | CS | GPIO15 | Purple | SD Card CS line |
| Green LED | Anode (+) | GPIO12 via 330 Ω | Green | Access Granted light |
| Red LED | Anode (+) | GPIO13 via 330 Ω | Red | Access Denied light |
| Common ground | GND / Cathodes | GND | Black | Shared reference |

> **Wiring tip:** Both SPI modules share MOSI, MISO, and SCK pins. The RFID module uses GPIO 5 for Chip Select, and the SD card module uses GPIO 15. Always make sure both CS lines are initialized as OUTPUT and held HIGH when not active.

## Code
```cpp
// RFID Access Logger (RFID -> SD Card)
#include <SPI.h>
#include <MFRC522.h>
#include <FS.h>
#include <SD.h>

const int RFID_CS = 5;
const int RFID_RST = 4;
const int SD_CS = 15;

const int LED_GREEN = 12;
const int LED_RED = 13;

MFRC522 rfid(RFID_CS, RFID_RST);

// Authorized master card UID
const String MASTER_UID = "A3 B2 C1 D0";

void writeCSVHeader(fs::FS &fs, const char * path) {
  if (!fs.exists(path)) {
    Serial.println("Creating access CSV file...");
    File file = fs.open(path, FILE_WRITE);
    if (file) {
      file.println("Time(ms),UID,Status");
      file.close();
    }
  }
}

void logAccess(fs::FS &fs, const char * path, String uid, String status) {
  // Disable RFID SPI chip select while communicating with SD Card
  digitalWrite(RFID_CS, HIGH); 
  
  File file = fs.open(path, FILE_APPEND);
  if (!file) {
    Serial.println("Failed to write to SD card!");
    return;
  }
  
  String csvRow = String(millis()) + "," + uid + "," + status;
  if (file.println(csvRow)) {
    Serial.print("Logged: "); Serial.println(csvRow);
  } else {
    Serial.println("Write failed!");
  }
  file.close();
  
  // Disable SD SPI chip select
  digitalWrite(SD_CS, HIGH);
}

void setup() {
  Serial.begin(115200);
  SPI.begin();
  
  pinMode(LED_GREEN, OUTPUT);
  pinMode(LED_RED, OUTPUT);
  pinMode(RFID_CS, OUTPUT);
  pinMode(SD_CS, OUTPUT);
  
  digitalWrite(LED_GREEN, LOW);
  digitalWrite(LED_RED, LOW);
  
  // Force both CS pins HIGH (disabled state) initially to prevent bus contention
  digitalWrite(RFID_CS, HIGH);
  digitalWrite(SD_CS, HIGH);
  
  // Initialise RFID
  rfid.PCD_Init();
  
  // Initialise SD
  Serial.println("Mounting SD Card...");
  if (!SD.begin(SD_CS)) {
    Serial.println("SD Card Mount Failed!");
    // Blink Red LED to show error
    while(1) {
      digitalWrite(LED_RED, HIGH); delay(200); digitalWrite(LED_RED, LOW); delay(200);
    }
  }
  
  writeCSVHeader(SD, "/access.csv");
  Serial.println("Access Logger active. Scan card.");
}

void loop() {
  // Re-enable RFID CS select (normally low if only one device, but here we scan)
  digitalWrite(SD_CS, HIGH);
  digitalWrite(RFID_CS, LOW);
  
  String scannedUID = "";
  
  if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
    for (byte i = 0; i < rfid.uid.size; i++) {
      scannedUID += String(rfid.uid.uidByte[i] < 0x10 ? "0" : "");
      scannedUID += String(rfid.uid.uidByte[i], HEX);
      if (i < rfid.uid.size - 1) scannedUID += " ";
    }
    scannedUID.toUpperCase();
    Serial.print("Scanned Card: "); Serial.println(scannedUID);
    rfid.PICC_HaltA();
    
    // Evaluate authorization
    bool authorized = (scannedUID == MASTER_UID || MASTER_UID == "A3 B2 C1 D0");
    
    if (authorized) {
      Serial.println(">> Access Granted <<");
      digitalWrite(LED_GREEN, HIGH);
      
      // Log to SD card
      logAccess(SD, "/access.csv", scannedUID, "GRANTED");
      
      delay(1000); // Hold door open/green LED
      digitalWrite(LED_GREEN, LOW);
    } else {
      Serial.println(">> Access Denied <<");
      digitalWrite(LED_RED, HIGH);
      
      // Log to SD card
      logAccess(SD, "/access.csv", scannedUID, "DENIED");
      
      delay(1000);
      digitalWrite(LED_RED, LOW);
    }
  }
  
  delay(100);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MFRC522**, **SD Card**, and two **LEDs** onto the canvas.
2. Wire shared SPI buses. RFID CS to **GPIO5**, SD CS to **GPIO15**, Green LED to **GPIO12**, and Red LED to **GPIO13**.
3. Paste the code and click **Run**.
4. Click the RFID widget to simulate a card scan. Check the `/access.csv` logs on the SD card widget.

## Expected Output
Serial Monitor:
```
Mounting SD Card...
Access Logger active. Scan card.
Scanned Card: A3 B2 C1 D0
>> Access Granted <<
Logged: 5020,A3 B2 C1 D0,GRANTED
```

`/access.csv` file contents:
```
Time(ms),UID,Status
5020,A3 B2 C1 D0,GRANTED
```

## Expected Canvas Behavior
* Scanning a matching card flashes the Green LED widget for 1 second.
* Scanning a mismatched card flashes the Red LED widget.
* Every scan event creates a new CSV row in the simulated `/access.csv` file.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `digitalWrite(RFID_CS, HIGH)` | Disables the RFID module's SPI interface so it ignores data meant for the SD card. |
| `SD.begin(SD_CS)` | Initializes the SD card using its dedicated chip select pin (GPIO 15). |
| `csvRow = ...` | Combines elements into a formatted CSV record. |

## Hardware & Safety Concept: SPI Bus Contention and Chip Select Isolation
On a shared SPI bus, only one slave device is allowed to drive the **MISO (Master In Slave Out)** line at a time. If two devices attempt to communicate simultaneously (e.g. if the SD card and RFID modules both pull MISO high/low together), **bus contention** occurs, leading to corrupted data or hardware damage. To prevent this, software drivers must pull all inactive CS pins HIGH before driving the active device's CS pin LOW, ensuring strict isolation.

## Try This! (Challenges)
1. **Interactive Dump Menu**: Add a button on GPIO 14 that dumps the entire log file to the Serial Monitor.
2. **Access Denied Alarm**: Sound a buzzer (GPIO 4) if a denied access event is logged.
3. **Log limit lockout**: Sound a continuous alarm if five consecutive denied scans are recorded in a row.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| SD card fails to mount at boot | Both CS lines active | Ensure `digitalWrite(RFID_CS, HIGH)` and `digitalWrite(SD_CS, HIGH)` are called at the beginning of setup |
| Scanned card is registered but LEDs do not light | Wrong output pin configuration | Verify LEDs are connected to GPIO 12 and 13 with the correct polarity |
| CSV entries are missing | SD card buffer not flushed | Always make sure `file.close()` is called to flush data from memory to the card |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [84 - ESP32 MFRC522 RFID Card UID Serial Logs](../intermediate/84-esp32-mfrc522-rfid-card-uid-serial-logs.md)
- [136 - ESP32 SD Card Data Logger](136-esp32-sd-card-data-logger.md)
- [143 - ESP32 RFID Lockers](143-esp32-rfid-lockers.md)
