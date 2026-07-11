# 137 - ESP32 SD Card Temperature Logger

Build a climate logging station that reads temperature and humidity from a DHT22 sensor every 5 seconds, appends the formatted data as a CSV record to an SPI SD card, and flashes a write status LED.

## Goal
Learn how to format sensor readings into CSV (Comma-Separated Values) strings, implement periodic non-blocking write schedules, and handle file system writes.

## What You Will Build
A DHT22 sensor is connected to GPIO 4. An SD card reader is connected via SPI (CS: 5). A status LED is on GPIO 12. Every 5 seconds, the ESP32 reads the climate values, formats them as `millis,temperature,humidity` and appends them to `/dht_log.csv` on the SD card, flashing the LED to confirm the write.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| MicroSD Card Shield (SPI) | `sd_card` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | DATA | GPIO4 | Yellow | Climate sensor input |
| DHT22 Sensor | VCC / GND | 3V3 / GND | Red / Black | Sensor power |
| SD Card Module | CS / SCK / MISO / MOSI | GPIO5 / 18 / 19 / 23 | Orange/Yellow/Green/Blue | SPI interface |
| SD Card Module | VCC / GND | 5V / GND | Red / Black | Power rails |
| Green LED | Anode (+) | GPIO12 via 330 Ω | Green | Status indicator |
| Green LED | Cathode (−) | GND | Black | Ground reference |

> **Wiring tip:** Share the 5V and GND lines for the SD card module and the LED. Connect the green LED's anode to GPIO 12 through a current-limiting resistor.

## Code
```cpp
// SD Card Temperature Logger (DHT22 -> CSV)
#include <FS.h>
#include <SD.h>
#include <SPI.h>
#include <DHT.h>

const int CS_PIN = 5;
const int DHT_PIN = 4;
const int LED_PIN = 12;

#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

unsigned long lastLogTime = 0;
const unsigned long LOG_INTERVAL_MS = 5000; // Log every 5 seconds

void writeCSVHeader(fs::FS &fs, const char * path) {
  // If file doesn't exist, create it and write headers
  if (!fs.exists(path)) {
    Serial.println("Creating CSV log file and writing headers...");
    File file = fs.open(path, FILE_WRITE);
    if (file) {
      file.println("Time(ms),Temp(C),Humidity(%)");
      file.close();
    }
  }
}

void appendToCSV(fs::FS &fs, const char * path, String dataRow) {
  File file = fs.open(path, FILE_APPEND);
  if (!file) {
    Serial.println("Error opening log file!");
    return;
  }
  
  if (file.println(dataRow)) {
    Serial.print("Logged: "); Serial.println(dataRow);
    // Flash status LED on successful write
    digitalWrite(LED_PIN, HIGH);
    delay(100);
    digitalWrite(LED_PIN, LOW);
  } else {
    Serial.println("Write failed!");
  }
  file.close();
}

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  
  dht.begin();
  
  Serial.println("Mounting SD Card...");
  if (!SD.begin(CS_PIN)) {
    Serial.println("SD Card Mount Failed!");
    while(1) { // Stop if SD card offline
      digitalWrite(LED_PIN, HIGH); delay(200); digitalWrite(LED_PIN, LOW); delay(200);
    }
  }
  
  // Write header row to dht_log.csv
  writeCSVHeader(SD, "/dht_log.csv");
  Serial.println("Logging active.");
}

void loop() {
  unsigned long now = millis();
  
  // Log on scheduled intervals (non-blocking)
  if (now - lastLogTime >= LOG_INTERVAL_MS) {
    float temp = dht.readTemperature();
    float hum = dht.readHumidity();
    
    if (!isnan(temp) && !isnan(hum)) {
      // Format CSV row: time_ms,temp,humidity
      String csvRow = String(now) + "," + String(temp, 1) + "," + String(hum, 1);
      
      appendToCSV(SD, "/dht_log.csv", csvRow);
    } else {
      Serial.println("Sensor read error!");
    }
    
    lastLogTime = now;
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22**, **SD Card**, and **LED** onto the canvas.
2. Wire DHT22 to **GPIO4**, SD to SPI (**CS: GPIO5**), and LED to **GPIO12**.
3. Paste the code and click **Run**.
4. Check the Serial Monitor. Watch the SD card write `Time(ms),Temp(C),Humidity(%)` values.
5. Watch the LED flash briefly every 5 seconds.

## Expected Output
Serial Monitor:
```
Mounting SD Card...
Creating CSV log file and writing headers...
Logging active.
Logged: 5000,24.5,45.2
Logged: 10000,24.6,45.1
```

CSV file contents (`/dht_log.csv`):
```
Time(ms),Temp(C),Humidity(%)
5000,24.5,45.2
10000,24.6,45.1
```

## Expected Canvas Behavior
* At boot, the SD card mounts.
* Every 5 seconds, the green LED flashes for 100 ms.
* The sensor's temperature and humidity are written to the virtual `/dht_log.csv` file.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `fs.exists(path)` | Verifies if the file already exists on the SD card to avoid duplicating headers. |
| `FILE_APPEND` | Opens the file in append mode, protecting old log entries from being overwritten. |
| `String(now) + "," + ...` | Formats data elements with comma delimiters for easy importing into Excel. |

## Hardware & Safety Concept: Comma-Separated Values (CSV) Structures
CSV (Comma-Separated Values) is a universal data format. By separating variables with commas and ending each record with a new line (`\n`), the files can be imported directly into spreadsheet software (such as Excel or Google Sheets) or parsed by Python scripts for plotting. Standard logging systems write data rows on fixed intervals to record environmental changes over time.

## Try This! (Challenges)
1. **Interactive Dump to Serial**: Add a button on GPIO 15. When pressed, read and print the entire CSV file to the USB Serial console.
2. **SD Write Error Alarm**: If the SD card is removed or fails, sound a warning on a buzzer connected to GPIO 13.
3. **Threshold-based Fast Log**: Increase the logging rate to once every 1 second if the temperature exceeds 30 °C.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED flashes rapidly at boot and hangs | SD card failed to initialize | Verify the SD card is formatted in FAT32; check SPI wiring |
| CSV shows "NaN" values | DHT22 read failure | Verify the DHT22 data pin is wired to GPIO 4 and sensor is powered |
| Log entries are lost when powered down | File not closed | Always make sure `file.close()` is executed after writing to flush the write buffer |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [136 - ESP32 SD Card Data Logger](136-esp32-sd-card-data-logger.md)
- [71 - ESP32 DHT22 Temp & Humidity Serial Logs](../intermediate/71-esp32-dht22-temp-humidity-serial-logs.md)
- [138 - ESP32 SD Card Current/Energy Logger](138-esp32-sd-card-current-energy-logger.md)
