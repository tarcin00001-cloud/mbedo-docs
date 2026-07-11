# 150 - ESP32 Dual-protocol Telemetry Logger

Build a high-performance environmental flight data recorder that samples data from an I2C sensor bus (MPU-6050 accelerometer/gyro and BMP180 barometric sensor) and logs it to an SPI SD card CSV file.

## Goal
Learn how to run multiple protocols (I2C and SPI) concurrently on the ESP32, coordinate sensor streams, format records, and verify write operations.

## What You Will Build
An MPU-6050 and BMP180 share the I2C bus (GPIO 21/22). An SD card reader uses SPI (CS: 15). A buzzer is connected to GPIO 4. Every 5 seconds, the ESP32 samples temperature, pressure, and 3-axis acceleration vectors, logging them to `/telemetry.csv` on the SD card, and double-beeps the buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| BMP180 Pressure Sensor | `bmp180` | Yes | Yes |
| MPU-6050 IMU Sensor | `mpu6050` | Yes | Yes |
| MicroSD Card Shield (SPI) | `sd_card` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 Sensor | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus (shared) |
| BMP180 Sensor | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus (shared) |
| SD Card Module | CS / SCK / MISO / MOSI | GPIO15 / 18 / 19 / 23 | Purple/Yellow/Green/Blue | SPI interface |
| Active Buzzer | VCC (+) | GPIO4 | Orange | Write success chime |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Standard I2C devices connect to GPIO 21 and 22. The SD Card Chip Select connects to GPIO 15. Make sure all logic grounds are tied to a common rail.

## Code
```cpp
// Dual-protocol Telemetry Logger (I2C Sensors -> SPI SD)
#include <Wire.h>
#include <SPI.h>
#include <FS.h>
#include <SD.h>
#include <Adafruit_BMP085.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>

const int SD_CS = 15;
const int BUZZER_PIN = 4;

Adafruit_BMP085 bmp;
Adafruit_MPU6050 mpu;

unsigned long lastLogTime = 0;
const unsigned long LOG_INTERVAL_MS = 5000; // Log every 5 seconds

void writeCSVHeader(fs::FS &fs, const char * path) {
  if (!fs.exists(path)) {
    Serial.println("Creating telemetry CSV log file...");
    File file = fs.open(path, FILE_WRITE);
    if (file) {
      file.println("Time(ms),Temp(C),Pres(hPa),AccX(m/s2),AccY(m/s2),AccZ(m/s2)");
      file.close();
    }
  }
}

void logTelemetry(fs::FS &fs, const char * path, float temp, float pres, float ax, float ay, float az) {
  File file = fs.open(path, FILE_APPEND);
  if (!file) {
    Serial.println("Failed to open log file!");
    return;
  }
  
  // Format CSV Row: Time, Temp, Pres, AccX, AccY, AccZ
  String csvRow = String(millis()) + "," + String(temp, 1) + "," + String(pres, 1) + "," + 
                  String(ax, 2) + "," + String(ay, 2) + "," + String(az, 2);
                  
  if (file.println(csvRow)) {
    Serial.print("Logged: "); Serial.println(csvRow);
    
    // Double beep on successful write
    digitalWrite(BUZZER_PIN, HIGH); delay(50);
    digitalWrite(BUZZER_PIN, LOW);  delay(50);
    digitalWrite(BUZZER_PIN, HIGH); delay(50);
    digitalWrite(BUZZER_PIN, LOW);
  } else {
    Serial.println("Write failed!");
  }
  file.close();
}

void setup() {
  Serial.begin(115200);
  
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  // Initialize I2C Bus
  Wire.begin();
  
  // Initialize BMP180
  if (!bmp.begin()) {
    Serial.println("BMP180 initialization failed!");
    while(1) {}
  }
  
  // Initialize MPU-6050
  if (!mpu.begin()) {
    Serial.println("MPU-6050 initialization failed!");
    while(1) {}
  }
  mpu.setAccelerometerRange(MPU6050_RANGE_8_G);
  
  // Initialize SD card
  Serial.println("Mounting SD Card...");
  if (!SD.begin(SD_CS)) {
    Serial.println("SD Card Mount Failed!");
    while(1) {}
  }
  
  writeCSVHeader(SD, "/telemetry.csv");
  Serial.println("Data logger active. Logging env & motion data.");
}

void loop() {
  unsigned long now = millis();
  
  if (now - lastLogTime >= LOG_INTERVAL_MS) {
    // 1. Read environmental data from BMP180
    float temp = bmp.readTemperature();
    float pressure = bmp.readPressure() / 100.0; // Convert Pa to hPa
    
    // 2. Read inertial data from MPU-6050
    sensors_event_t a, g, temp_mpu;
    mpu.getEvent(&a, &g, &temp_mpu);
    
    float ax = a.acceleration.x;
    float ay = a.acceleration.y;
    float az = a.acceleration.z;
    
    // 3. Log data to SD Card CSV
    logTelemetry(SD, "/telemetry.csv", temp, pressure, ax, ay, az);
    
    lastLogTime = now;
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **BMP180**, **MPU-6050**, **SD Card**, and **Buzzer** onto the canvas.
2. Wire both I2C sensors to **GPIO21/GPIO22**, SD CS to **GPIO15**, and Buzzer to **GPIO4**.
3. Paste the code and click **Run**.
4. Adjust sensor values on the simulated widgets. Watch the buzzer beep every 5 seconds and log coordinates to `/telemetry.csv`.

## Expected Output
Serial Monitor:
```
Mounting SD Card...
Data logger active. Logging env & motion data.
Logged: 5000,24.5,1013.2,-0.12,0.45,9.80
Logged: 10000,24.6,1013.1,-0.15,0.42,9.78
```

`/telemetry.csv` file contents:
```
Time(ms),Temp(C),Pres(hPa),AccX(m/s2),AccY(m/s2),AccZ(m/s2)
5000,24.5,1013.2,-0.12,0.45,9.80
10000,24.6,1013.1,-0.15,0.42,9.78
```

## Expected Canvas Behavior
* At boot, the SD card mounts.
* Every 5 seconds, the buzzer widget flashes green (beeps).
* Adjusting the temperature, pressure, or acceleration sliders on the simulated widgets changes the values logged to the CSV file.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Wire.begin()` | Starts the I2C master driver on the standard ESP32 pins. |
| `bmp.readPressure() / 100.0` | Converts Pascal barometric readings to hectopascal (hPa). |
| `logTelemetry(...)` | Aggregates all sensor data points into a single CSV row. |

## Hardware & Safety Concept: Multi-protocol Buses and SPI/I2C Separation
Modern telemetry loggers use separate protocols for different roles:
1. **I2C Bus**: Low-speed serial bus (100–400 kHz) for reading small sensor registers. It uses open-drain lines with pull-up resistors, making it easy to add multiple sensors in parallel.
2. **SPI Bus**: High-speed serial bus (10–40 MHz) with push-pull lines, making it ideal for writing large data blocks quickly (e.g. to SD card sectors).
Running these protocols on separate pins allows the ESP32 to query sensors and write files concurrently without bus contention.

## Try This! (Challenges)
1. **Flight Fall Trigger**: If acceleration drops below 3.0 \(m/s^2\) (simulating freefall), log data at 50 Hz (once every 20 ms) to capture high-resolution crash details.
2. **Interactive Dump button**: Add a button on GPIO 14 that dumps the entire log file to the Serial Monitor.
3. **Altitude warning indicator**: Sound a warning siren if the pressure altitude changes by more than 10 meters within 1 second.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| SD card fails to initialize | CS line shared with other SPI devices | Ensure the SD CS pin is wired to GPIO 15 and initialized correctly |
| BMP180 or MPU-6050 fails to start | I2C address conflict or loose wire | Run an I2C scanner script to verify both devices are detected on the bus |
| Buzzer does not sound | Buzzer pin misconfigured | Confirm the active buzzer is connected to GPIO 4 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [136 - ESP32 SD Card Data Logger](136-esp32-sd-card-data-logger.md)
- [139 - ESP32 Multi-Sensor Environmental HUD](139-esp32-multi-sensor-environmental-hud.md)
- [147 - ESP32 GPS Velocity Logger](147-esp32-gps-velocity-logger.md)
