# 136 - SD Card Data Logger

Initialize an SD card module on standard SPI pins using the VEGA ARIES v3 board and log a simple "Hello World" message to verify write functionality.

## Goal
Learn how to interface the VEGA ARIES v3 board with an SPI-based SD Card module in MbedO interpreted mode, mount the filesystem, open a file, write a text payload, and safely close the file to avoid data corruption.

## What You Will Build
A basic SPI telemetry logger. When the simulation starts, the controller initializes the SD Card module via SPI. It then opens a text file named `log.txt`, writes the line `Hello World`, closes the file, and prints the logging status to the Serial Monitor. A state variable prevents the code from writing repeatedly.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Micro SD Card Module | `sd_card` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| SD Card Module | VCC | 5V | Red | Primary 5V power |
| SD Card Module | GND | GND | Black | Ground reference |
| SD Card Module | CS | GP10 | Blue | SPI Slave Select (CS) |
| SD Card Module | MOSI | GP11 | Yellow | SPI Master Out Slave In |
| SD Card Module | MISO | GP12 | Green | SPI Master In Slave Out |
| SD Card Module | SCK | GP13 | White | SPI Serial Clock |

> **Wiring tip:** Keep SPI connection lines as short as possible to reduce signal interference. Make sure you use the designated hardware SPI0 pins on the ARIES board (GP10 to GP13).

## Code
```cpp
#include <SPI.h>
#include <SD.h>

const int chipSelect = 10;
bool isLogged = false;

void setup() {
  Serial.begin(115200);
  
  // Wait for serial port to connect
  delay(1000);
  Serial.println("Initializing SD card...");

  // Initialize standard SPI SD card
  if (!SD.begin(chipSelect)) {
    Serial.println("Card initialization failed!");
  } else {
    Serial.println("Card initialized successfully.");
  }
}

void loop() {
  if (!isLogged) {
    // Open the file for writing
    File dataFile = SD.open("log.txt", FILE_WRITE);
    
    // If the file is available, write to it
    if (dataFile) {
      dataFile.println("Hello World");
      dataFile.close();
      Serial.println("Data logged: Hello World");
      isLogged = true;
    } else {
      Serial.println("Error opening log.txt");
      // Keep trying in the next loops if it fails
    }
  }
  
  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board and the **SD Card Module** onto the canvas.
2. Connect the SD Card module pins: **VCC** to **5V**, **GND** to **GND**, **CS** to **GP10**, **MOSI** to **GP11**, **MISO** to **GP12**, and **SCK** to **GP13**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the program.
6. Check the Serial Monitor for confirmation messages.

## Expected Output
Serial Monitor:
```
System Initialized.
Initializing SD card...
Card initialized successfully.
Data logged: Hello World
```

## Expected Canvas Behavior
* The SD Card Module widget lights up to indicate successful SPI initialization.
* The Serial Monitor prints the initialization and log success messages within the first few seconds of simulation.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `SD.begin(chipSelect)` | Configures the SPI interface and mounts the FAT filesystem on the SD card using CS pin GP10. |
| `SD.open("log.txt", FILE_WRITE)` | Opens `log.txt` with write access, creating it if it does not already exist. |
| `dataFile.println(...)` | Appends a text line to the open file buffer on the SD card. |
| `dataFile.close()` | Flushes any remaining bytes from the microcontroller buffer to the physical SD card and releases the file handle. |
| `isLogged = true` | Latches the state variable to prevent repeated writes to the SD card during loop execution. |

## Hardware & Safety Concept
* **FAT Filesystem Overhead**: SD cards require a FAT16 or FAT32 file allocation system. The microcontroller library processes sectors in 512-byte blocks. If you do not call `.close()`, the file allocation table is not updated and data is lost.
* **SPI Signal Integrity**: High clock rates can cause signal degradation on breadboards. Keep wire runs short and avoid sharing SPI pins with noisy analog signals or high-power switching elements.

## Try This! (Challenges)
1. **Append Timestamp**: Read `millis()` and write the time in milliseconds alongside "Hello World" (e.g., `Hello World at 2350ms`).
2. **Log Reset Counter**: Retrieve the log file, read how many times it was written, and append the run index.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Card initialization failed! | Missing or loose SPI wiring | Check the CS pin on GP10, MOSI on GP11, MISO on GP12, and SCK on GP13. |
| Error opening log.txt | SD Card not formatted correctly | Ensure the virtual or physical SD card is formatted with FAT16 or FAT32. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [137 - SD Card Temperature Logger](137-sd-card-temperature-logger.md) (Next project)
