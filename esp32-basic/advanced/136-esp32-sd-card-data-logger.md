# 136 - ESP32 SD Card Data Logger

Build an SPI SD card reader interface that initializes the storage card, writes test log strings to a text file, and reads them back to verify data integrity.

## Goal
Learn how to wire and program SPI SD card modules, handle file systems using the `SD` and `FS` libraries, and handle file read/write operations.

## What You Will Build
An SD card module is connected to the ESP32 using the SPI interface (MOSI: 23, MISO: 19, SCK: 18, CS: 5). The code mounts the file system at boot, creates/appends a line to a file named `/log.txt`, reads back the entire file, and prints the contents to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MicroSD Card Shield (SPI) | `sd_card` | Yes | Yes |
| MicroSD Card (FAT16/FAT32) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| SD Card Module | CS | GPIO5 | Orange | Chip Select |
| SD Card Module | SCK | GPIO18 | Yellow | SPI Clock |
| SD Card Module | MISO | GPIO19 | Green | Master In Slave Out |
| SD Card Module | MOSI | GPIO23 | Blue | Master Out Slave In |
| SD Card Module | VCC | 5V | Red | Power (usually 5V input) |
| SD Card Module | GND | GND | Black | Ground reference |

> **Wiring tip:** Connect the SPI lines to the standard ESP32 VSPI pins. CS goes to GPIO 5. The SD card module must be formatted as FAT16 or FAT32.

## Code
```cpp
// SD Card Data Logger (SPI Initialization)
#include <FS.h>
#include <SD.h>
#include <SPI.h>

const int CS_PIN = 5;

void appendFile(fs::FS &fs, const char * path, const char * message) {
  Serial.print("Appending to file: "); Serial.println(path);

  File file = fs.open(path, FILE_APPEND);
  if(!file) {
    Serial.println("Failed to open file for appending");
    return;
  }
  if(file.print(message)) {
    Serial.println("Message appended successfully");
  } else {
    Serial.println("Append failed");
  }
  file.close();
}

void readFile(fs::FS &fs, const char * path) {
  Serial.print("Reading file: "); Serial.println(path);

  File file = fs.open(path);
  if(!file) {
    Serial.println("Failed to open file for reading");
    return;
  }

  Serial.println("--- FILE CONTENT START ---");
  while(file.available()) {
    Serial.write(file.read());
  }
  Serial.println("--- FILE CONTENT END ---");
  file.close();
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("Mounting SD Card...");
  
  if(!SD.begin(CS_PIN)) {
    Serial.println("SD Card Mount Failed! Check connections.");
    return;
  }
  
  uint8_t cardType = SD.cardType();
  if(cardType == CARD_NONE) {
    Serial.println("No SD card attached!");
    return;
  }
  
  Serial.print("SD Card Type: ");
  if(cardType == CARD_MMC) Serial.println("MMC");
  else if(cardType == CARD_SD) Serial.println("SDSC");
  else if(cardType == CARD_SDHC) Serial.println("SDHC");
  else Serial.println("UNKNOWN");

  // Get and print card size
  uint64_t cardSize = SD.cardSize() / (1024 * 1024);
  Serial.printf("SD Card Size: %lluMB\n", cardSize);

  // Write/append test data
  appendFile(SD, "/log.txt", "ESP32 Boot Log Event\n");
  
  // Read back and print file contents
  readFile(SD, "/log.txt");
}

void loop() {
  // Idle
  delay(1000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **SD Card** onto the canvas.
2. Wire SCK to **GPIO18**, MISO to **GPIO19**, MOSI to **GPIO23**, and CS to **GPIO5**.
3. Paste the code and click **Run**.
4. Check the Serial Monitor. Watch the SD card mount, size display, write, and dump file contents.

## Expected Output
Serial Monitor:
```
Mounting SD Card...
SD Card Type: SDHC
SD Card Size: 15190MB
Appending to file: /log.txt
Message appended successfully
Reading file: /log.txt
--- FILE CONTENT START ---
ESP32 Boot Log Event
--- FILE CONTENT END ---
```

## Expected Canvas Behavior
* The SD card widget mounts and exchanges SPI data frames, showing active indicator blinks.
* Writing completes successfully.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `SD.begin(CS_PIN)` | Configures VSPI pins and initializes communication with the SD card. |
| `fs.open(path, FILE_APPEND)` | Opens the target file in append mode (adding to the end without wiping). |
| `file.read()` | Reads bytes sequentially from the SD card storage flash. |

## Hardware & Safety Concept: SPI CS Pins and FAT File Systems
SPI (Serial Peripheral Interface) is a bus protocol sharing Clock (SCK), Data In (MISO), and Data Out (MOSI) lines between multiple devices. The **Chip Select (CS)** pin is dedicated to each device. Driving a device's CS line LOW enables it, while CS HIGH makes it ignore the bus. SD cards require formatting in FAT16/FAT32 formats so the library can navigate directories and sectors. ExFAT is not supported by standard Arduino SD libraries.

## Try This! (Challenges)
1. **Dynamic File Creator**: Add a button on GPIO 4 that creates a new unique text file (e.g. `/log2.txt`) every time it is pressed.
2. **Clear Log file**: Add a command that deletes the file using `SD.remove("/log.txt")` when a button is held down.
3. **Card space monitor**: Read and display the remaining free space of the SD card using `SD.usedBytes()`.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| SD Card Mount Failed | Card formatted as exFAT or bad wiring | Format the card as FAT32; check SPI connections |
| Write fails | Card write-protect switch locked | Check the lock slider switch on the side of the SD card adapter |
| Card type is UNKNOWN | Card not fully inserted | Re-insert the card into the shield slot |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [137 - ESP32 SD Card Temperature Logger](137-esp32-sd-card-temperature-logger.md)
- [138 - ESP32 SD Card Current/Energy Logger](138-esp32-sd-card-current-energy-logger.md)
- [84 - ESP32 MFRC522 RFID Card UID Serial Logs](../intermediate/84-esp32-mfrc522-rfid-card-uid-serial-logs.md)
