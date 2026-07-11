# 169 - ESP32 Multi-zone Temperature Logger

Build an environmental monitoring station that reads three DS18B20 digital temperature sensors sharing a single OneWire bus, logs the readings to an SPI SD card, and displays them on a cycling LCD.

## Goal
Learn how to address and query multiple digital sensors sharing a single OneWire bus, manage SD card CSV logs, and build multi-page LCD telemetry menus.

## What You Will Build
Three DS18B20 temperature sensors are connected to the same OneWire data bus on GPIO 4. An SD card reader is on SPI (CS: 15). A status LED is on GPIO 12, and a 16x2 LCD on I2C. The ESP32 queries all three sensors, appends their temperatures to `/multizone.csv` every 10 seconds, and cycles the LCD to display the readings.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DS18B20 Temperature Sensors (3) | `ds18b20` | Yes | Yes |
| MicroSD Card Shield (SPI) | `sd_card` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| 4.7 kΩ Resistor (OneWire pull-up) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS18B20 Sensors (All) | DQ (Data) | GPIO4 | Green | Shared OneWire bus |
| 4.7 kΩ Resistor | Leg 1 / Leg 2 | GPIO4 / 3V3 | White | Bus pull-up resistor |
| DS18B20 Sensors (All) | VCC / GND | 3V3 / GND | Red / Black | Shared power rails |
| SD Card Module | CS / SCK / MISO / MOSI | GPIO15 / 18 / 19 / 23 | Purple/Yellow/Green/Blue | SPI interface |
| Green LED | Anode (+) | GPIO12 via 330 Ω | Green | Log confirmation LED |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |

> **Wiring tip:** The three DS18B20 sensors must connect in parallel to the same data line (GPIO 4). A single 4.7 kΩ pull-up resistor is shared at the breadboard connection point.

## Code
```cpp
// Multi-zone Temperature logger (3x DS18B20 -> SD Card)
#include <OneWire.h>
#include <DallasTemperature.h>
#include <SPI.h>
#include <FS.h>
#include <SD.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int ONE_WIRE_BUS = 4;
const int LED_PIN = 12;
const int SD_CS = 15;

OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);
LiquidCrystal_I2C lcd(0x27, 16, 2);

int deviceCount = 0;
unsigned long lastLogTime = 0;
const unsigned long LOG_INTERVAL_MS = 10000; // Log every 10 seconds

unsigned long lastPageChange = 0;
int lcdPage = 0;

float temp0 = 0.0;
float temp1 = 0.0;
float temp2 = 0.0;

void writeCSVHeader(fs::FS &fs, const char * path) {
  if (!fs.exists(path)) {
    File file = fs.open(path, FILE_WRITE);
    if (file) {
      file.println("Time(s),Zone0(C),Zone1(C),Zone2(C)");
      file.close();
    }
  }
}

void logTemperature(fs::FS &fs, const char * path, float t0, float t1, float t2) {
  File file = fs.open(path, FILE_APPEND);
  if (!file) {
    Serial.println("Failed to open log file!");
    return;
  }
  
  String csvRow = String(millis() / 1000) + "," + String(t0, 1) + "," + String(t1, 1) + "," + String(t2, 1);
  if (file.println(csvRow)) {
    Serial.print("Logged: "); Serial.println(csvRow);
    
    // Flash status LED
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
  
  // Start OneWire sensors
  sensors.begin();
  deviceCount = sensors.getDeviceCount();
  Serial.print("Found "); Serial.print(deviceCount); Serial.println(" DS18B20 devices.");
  
  // Start LCD
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Zone Temp Logger");
  lcd.setCursor(0, 1);
  lcd.print("Init SD Card...");
  
  // Start SD Card
  if (!SD.begin(SD_CS)) {
    Serial.println("SD Card Mount Failed!");
    lcd.setCursor(0, 1);
    lcd.print("SD CARD ERROR   ");
    while(1) {}
  }
  
  writeCSVHeader(SD, "/multizone.csv");
  delay(1500);
  lcd.clear();
}

void loop() {
  unsigned long now = millis();
  
  // 1. Read all sensors and log data every 10 seconds
  if (now - lastLogTime >= LOG_INTERVAL_MS) {
    sensors.requestTemperatures();
    
    // Read by index on the shared wire bus
    temp0 = sensors.getTempCByIndex(0);
    temp1 = sensors.getTempCByIndex(1);
    temp2 = sensors.getTempCByIndex(2);
    
    // Check for errors (disconnected sensors)
    if (temp0 == DEVICE_DISCONNECTED_C) temp0 = 0.0;
    if (temp1 == DEVICE_DISCONNECTED_C) temp1 = 0.0;
    if (temp2 == DEVICE_DISCONNECTED_C) temp2 = 0.0;
    
    logTemperature(SD, "/multizone.csv", temp0, temp1, temp2);
    
    lastLogTime = now;
  }
  
  // 2. Cycle LCD display screens every 4 seconds
  if (now - lastPageChange >= 4000) {
    lcdPage = (lcdPage + 1) % 2;
    lcd.clear();
    lastPageChange = now;
  }
  
  if (lcdPage == 0) {
    lcd.setCursor(0, 0);
    lcd.print("Zone 0: "); lcd.print(temp0, 1); lcd.print(" C ");
    lcd.setCursor(0, 1);
    lcd.print("Zone 1: "); lcd.print(temp1, 1); lcd.print(" C ");
  } else {
    lcd.setCursor(0, 0);
    lcd.print("Zone 2: "); lcd.print(temp2, 1); lcd.print(" C ");
    lcd.setCursor(0, 1);
    lcd.print("Status: Logged  ");
  }
  
  delay(50);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, three **DS18B20 Sensors**, **SD Card**, and **16x2 I2C LCD** onto the canvas.
2. Wire all three DS18B20 to **GPIO4**, SD CS to **GPIO15**, LED to **GPIO12**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Adjust temperature values on the simulated sensor widgets.
5. Watch the Green LED flash every 10 seconds as it logs the multi-zone readings to `/multizone.csv`.

## Expected Output
Serial Monitor:
```
Found 3 DS18B20 devices.
Logged: 10,24.5,26.2,22.8
Logged: 20,24.6,26.5,22.9
```

`/multizone.csv` file contents:
```
Time(s),Zone0(C),Zone1(C),Zone2(C)
10,24.5,26.2,22.8
20,24.6,26.5,22.9
```

## Expected Canvas Behavior
* At boot, the serial output prints "Found 3 DS18B20 devices".
* Adjusting the sliders on the three temperature sensors updates the LCD values.
* The LCD cycles between Page 0 (Zones 0/1) and Page 1 (Zone 2/Status) every 4 seconds.
* The LED flashes green every 10 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `sensors.getDeviceCount()` | Scans the OneWire bus on startup to count how many sensors are connected in parallel. |
| `sensors.requestTemperatures()` | Broadcasts a command to all sensors on the bus to convert temperatures. |
| `sensors.getTempCByIndex(0)` | Reads the temperature value from the sensor at index 0. |

## Hardware & Safety Concept: OneWire Bus Addressing
The Dallas OneWire protocol allows connecting multiple sensors in parallel using a single digital GPIO pin. Each DS18B20 sensor has a unique, factory-programmed 64-bit ROM registration address (like `0x28 0xFF 0x4B 0x44 0x05 0x00 0x00 0x1C`). The library communicates with each sensor individually using these addresses, allowing a single microcontroller pin to monitor dozens of separate thermal zones.

## Try This! (Challenges)
1. **Critical Overheat Alarm**: Add a buzzer on GPIO 14 that sounds a siren if any of the temperature zones exceed 45 °C.
2. **Dynamic sensor scanner**: Print the 64-bit ROM address of all connected sensors to the Serial Monitor at boot.
3. **Difference calculation HUD**: Add a third LCD page displaying the temperature difference between Zone 1 and Zone 2.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Found 0 DS18B20 devices | Missing pull-up resistor | Verify the 4.7 kΩ pull-up resistor is installed between the data line (GPIO 4) and 3.3V |
| All temperatures read -127 °C | Sensor disconnected | Check that all three sensors share ground, VCC, and data lines |
| SD card fails to mount | Pin mapping conflict | Confirm SD CS pin is wired to GPIO 15, not GPIO 5 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [81 - ESP32 DS18B20 OneWire Temp Sensor Serial Logs](../intermediate/81-esp32-ds18b20-onewire-temp-sensor-serial-logs.md)
- [137 - ESP32 SD Card Temperature Logger](137-esp32-sd-card-temperature-logger.md)
- [150 - ESP32 Dual-protocol Telemetry Logger](150-esp32-dual-protocol-telemetry-logger.md)
