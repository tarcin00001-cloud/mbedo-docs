# 137 - SD Card Temperature Logger

Read temperature and humidity from a DHT22 sensor and log the readings to a CSV file on an SD card at regular intervals using the VEGA ARIES v3 board.

## Goal
Learn how to interface multiple SPI and single-bus sensors simultaneously, format data into standard comma-separated values (CSV), log data periodically using non-blocking timers, and handle potential file access issues in interpreted C++.

## What You Will Build
A standalone temperature logging station. Every 2 seconds, the controller reads the temperature and relative humidity from a DHT22 sensor connected to GPIO 12. It appends the current uptime (in milliseconds), temperature (in Celsius), and humidity (in %) to a CSV file named `temp.csv` on the SD card, and prints the same data to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Micro SD Card Module | `sd_card` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| SD Card Module | VCC | 5V | Red | Power supply |
| SD Card Module | GND | GND | Black | Ground reference |
| SD Card Module | CS | GP10 | Blue | SPI Slave Select |
| SD Card Module | MOSI | GP11 | Yellow | SPI Master Out |
| SD Card Module | MISO | GP12 | Green | SPI Master In (Shared) |
| SD Card Module | SCK | GP13 | White | SPI Serial Clock |
| DHT22 Sensor | VCC | 3V3 | Orange | Power supply (3.3V) |
| DHT22 Sensor | SDA (Data) | GPIO 12 | Purple | Digital Single-Bus Data (Shared GP12) |
| DHT22 Sensor | GND | GND | Grey | Ground reference |

> **Wiring tip:** The DHT22 single-bus data line uses GPIO 12. Since GP12 is also used by the SPI MISO line, ensure that the simulation handles sharing or connect the sensor correctly according to the board's setup instructions.

## Code
```cpp
#include <SPI.h>
#include <SD.h>
#include <DHT.h>

#define DHTPIN 12
#define DHTTYPE DHT22

DHT dht(DHTPIN, DHTTYPE);
const int chipSelect = 10;
unsigned long lastLogTime = 0;

void setup() {
  Serial.begin(115200);
  
  // Wait for peripherals to settle
  delay(1000);
  
  dht.begin();
  Serial.println("Initializing SD card...");

  if (!SD.begin(chipSelect)) {
    Serial.println("SD Card initialization failed!");
  } else {
    Serial.println("SD Card initialized successfully.");
  }
}

void loop() {
  unsigned long currentTime = millis();

  // Log temperature and humidity every 2000 ms (2 seconds)
  if (currentTime - lastLogTime >= 2000) {
    float temp = dht.readTemperature();
    float hum = dht.readHumidity();

    // Check if readings are valid
    if (isnan(temp) || isnan(hum)) {
      Serial.println("Failed to read from DHT sensor!");
    } else {
      // Open file for appending
      File dataFile = SD.open("temp.csv", FILE_WRITE);

      if (dataFile) {
        dataFile.print(currentTime);
        dataFile.print(",");
        dataFile.print(temp);
        dataFile.print(",");
        dataFile.println(hum);
        dataFile.close();

        // Print confirmation to serial monitor
        Serial.print("Time: ");
        Serial.print(currentTime);
        Serial.print(" ms | Temp: ");
        Serial.print(temp);
        Serial.print(" C | Hum: ");
        Serial.print(hum);
        Serial.println(" %");
      } else {
        Serial.println("Error opening temp.csv");
      }
    }

    lastLogTime = currentTime;
  }
  
  delay(10); // Short debounce delay
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, **SD Card Module**, and **DHT22 Sensor** onto the canvas.
2. Connect the SD Card module pins: **VCC** to **5V**, **GND** to **GND**, **CS** to **GP10**, **MOSI** to **GP11**, **MISO** to **GP12**, and **SCK** to **GP13**.
3. Connect the DHT22 pins: **VCC** to **3V3**, **GND** to **GND**, and **SDA** to **GPIO 12**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the program.
7. Modify the temperature/humidity sliders on the DHT22 widget and observe the logs.

## Expected Output
Serial Monitor:
```
System Initialized.
Initializing SD card...
SD Card initialized successfully.
Time: 2000 ms | Temp: 24.50 C | Hum: 55.20 %
Time: 4000 ms | Temp: 25.10 C | Hum: 54.80 %
```

## Expected Canvas Behavior
* The DHT22 sensor readings update dynamically when sliders are manipulated.
* The SD Card Module logs the entries as commas-separated rows, simulating file growth.
* Output messages print to the Serial Monitor every 2 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `dht.begin()` | Prepares the DHT library protocols and configures the single-wire pin GP12. |
| `currentTime - lastLogTime >= 2000` | Calculates elapsed time dynamically using `millis()` to enforce a 2-second rate limit without blocking. |
| `dht.readTemperature()` | Queries the DHT22 sensor to return the current temperature reading as a floating-point number. |
| `SD.open("temp.csv", FILE_WRITE)` | Opens `temp.csv` in append mode. New logs are added to the end of the file. |
| `dataFile.print(temp) + dataFile.print(",")` | Formats data elements with separating commas to comply with CSV structures. |

## Hardware & Safety Concept
* **Sensors and SPI Sharing**: Single-wire protocols like the DHT22 rely on strict microsecond timing. Sharing pins with hardware SPI buses like MISO (GP12) can occasionally cause noise or read failures on physical hardware. In real builds, it is recommended to move the DHT22 to an independent GPIO.
* **Thermal Mass**: DHT22 sensors have a thermal mass that prevents instantaneous readings. Sampling faster than once every 2 seconds will return stale or inaccurate values.

## Try This! (Challenges)
1. **Convert to Fahrenheit**: Add code inside loop to convert Celsius to Fahrenheit and write both columns to the SD card.
2. **Alarm Logging**: Add a warning flag column (`0` or `1`). If the temperature exceeds 30 degrees, write a `1` in the warning column, otherwise write `0`.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Failed to read from DHT sensor! | Bad signal connection or speed issue | Verify that the DHT22 VCC is on 3.3V and data is connected to GP12. |
| SD Card fails on startup | Multiple SPI modules competing | Double check that the CS pin GP10 is selected and matches your library initialization. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [136 - SD Card Data Logger](136-sd-card-data-logger.md) (Previous project)
- [138 - SD Card Current/Energy Logger](138-sd-card-current-energy-logger.md) (Next project)
