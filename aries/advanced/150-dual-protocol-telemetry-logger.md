# 150 - Dual-protocol Telemetry Logger

Read environmental sensor data from multiple I2C devices on I2C0, then log every reading as a timestamped CSV row to a microSD card connected via the SPI0 bus — combining I2C and SPI protocols in a single firmware to produce a persistent multi-channel telemetry archive.

## Goal

Learn how to orchestrate two fundamentally different embedded communication protocols simultaneously: poll an MPU-6050 IMU (acceleration, temperature) and a BMP280 pressure/temperature sensor over I2C, then write the combined sensor snapshot to a CSV file on an SPI-connected SD card — building the foundation of a real-world environmental data logger or flight recorder.

## What You Will Build

The MPU-6050 IMU (I2C address `0x68`) provides three-axis acceleration and die temperature. The BMP280 (I2C address `0x76`) provides barometric pressure and ambient temperature. Every 500 ms the firmware reads both sensors over I2C0, assembles a CSV line (`SampleNo,Ax_g,Ay_g,Az_g,TempIMU_C,Pressure_Pa,TempBaro_C`), and appends it to `TELEM.CSV` on the SPI SD card. All data is also echoed to the Serial Monitor.

## Parts Needed

| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MPU-6050 IMU Module | `mpu6050` | Yes | Yes |
| BMP280 Barometric Sensor | `bmp280` | Yes | Yes |
| MicroSD Card Module (SPI) | `sdcard` | Yes | Yes |
| MicroSD Card (FAT32) | — | No | Yes |
| Jumper Wires | — | No | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 | VCC | 3V3 | Red | 3.3 V IMU power |
| MPU-6050 | GND | GND | Black | Common ground |
| MPU-6050 | SDA | GPIO 0 (I2C0 SDA) | Orange | I2C data bus |
| MPU-6050 | SCL | GPIO 1 (I2C0 SCL) | Yellow | I2C clock bus |
| MPU-6050 | AD0 | GND | Black | I2C address = 0x68 |
| BMP280 | VCC | 3V3 | Red | 3.3 V sensor power |
| BMP280 | GND | GND | Black | Common ground |
| BMP280 | SDA | GPIO 0 (I2C0 SDA) | Orange | Shared I2C SDA |
| BMP280 | SCL | GPIO 1 (I2C0 SCL) | Yellow | Shared I2C SCL |
| BMP280 | SDO | GND | Black | I2C address = 0x76 (SDO to VCC = 0x77) |
| SD Card Module | VCC | 3V3 | Red | SD module power |
| SD Card Module | GND | GND | Black | Common ground |
| SD Card Module | SCK | GPIO 6 (SPI0 SCK) | Yellow | SPI clock |
| SD Card Module | MOSI | GPIO 7 (SPI0 TX) | Blue | SPI data to SD |
| SD Card Module | MISO | GPIO 4 (SPI0 RX) | Green | SPI data from SD |
| SD Card Module | CS | GPIO 9 | White | SPI chip-select for SD |

> **Wiring tip:** The I2C bus (GPIO 0/1) and SPI bus (GPIO 4/6/7/9) are completely independent — they can be active simultaneously with no conflicts. MPU-6050 AD0 to GND sets its address to `0x68`; BMP280 SDO to GND sets its address to `0x76`. Both are on the same I2C0 bus and queried sequentially. Ensure 4.7 kΩ pull-up resistors on SDA and SCL. Pre-format the SD card as FAT32.

## Code

```cpp
// 150 - Dual-protocol Telemetry Logger
// I2C: MPU-6050 (accel + temp) + BMP280 (pressure + temp) -> SPI SD Card CSV
#include <Wire.h>
#include <SPI.h>
#include <SD.h>
#include <Adafruit_BMP280.h>

// ---- SPI SD Card ----
#define SD_CS_PIN 9
int sdReady = 0;

// ---- I2C Addresses ----
#define MPU_ADDR 0x68

// ---- MPU-6050 Registers ----
#define MPU_PWR_MGMT_1 0x6B
#define MPU_ACCEL_XOUT 0x3B
#define MPU_TEMP_OUT   0x41  // 2 bytes: temp MSB, temp LSB

// ---- BMP280 ----
Adafruit_BMP280 bmp;
int bmpReady = 0;
int mpuReady = 0;

// ---- Accelerometer readings ----
int   axRaw = 0;
int   ayRaw = 0;
int   azRaw = 0;
float axG   = 0.0;
float ayG   = 0.0;
float azG   = 0.0;

// ---- MPU-6050 die temperature ----
int   mpuTempRaw = 0;
float mpuTempC   = 0.0;

// ---- BMP280 readings ----
float bmpPressurePa = 0.0;
float bmpTempC      = 0.0;

// ---- I2C byte helpers ----
int rawMsb = 0;
int rawLsb = 0;

// ---- Logging state ----
int sampleNo = 0;
unsigned long lastLogMs = 0;
unsigned long nowMs     = 0;

// ---- CSV line ----
String csvLine = "";

void setup() {
  Serial.begin(115200);
  Serial.println("=== Dual-protocol Telemetry Logger ===");
  Serial.println("I2C sensors: MPU-6050 + BMP280");
  Serial.println("SPI storage: SD Card -> TELEM.CSV");
  Serial.println("-------------------------------------------");

  Wire.begin();
  SPI.begin();

  // ---- Wake MPU-6050 ----
  Wire.beginTransmission(MPU_ADDR);
  Wire.write(MPU_PWR_MGMT_1);
  Wire.write(0x00);  // Clear sleep bit
  int mpuErr = Wire.endTransmission();
  if (mpuErr == 0) {
    mpuReady = 1;
    Serial.println("MPU-6050 ready at 0x68.");
  } else {
    mpuReady = 0;
    Serial.print("ERROR: MPU-6050 not found. I2C error code: ");
    Serial.println(mpuErr);
  }

  // ---- Initialise BMP280 ----
  if (!bmp.begin(0x76)) {
    bmpReady = 0;
    Serial.println("ERROR: BMP280 not found at 0x76. Check SDO pin.");
  } else {
    bmpReady = 1;
    // Set oversampling and filter for stable readings
    bmp.setSampling(Adafruit_BMP280::MODE_NORMAL,
                    Adafruit_BMP280::SAMPLING_X2,   // temperature
                    Adafruit_BMP280::SAMPLING_X16,  // pressure
                    Adafruit_BMP280::FILTER_X16,
                    Adafruit_BMP280::STANDBY_MS_500);
    Serial.println("BMP280 ready at 0x76.");
  }

  // ---- Initialise SD Card ----
  if (!SD.begin(SD_CS_PIN)) {
    sdReady = 0;
    Serial.println("ERROR: SD card initialisation failed. Check wiring.");
  } else {
    sdReady = 1;
    Serial.println("SD card ready.");

    if (!SD.exists("TELEM.CSV")) {
      File f = SD.open("TELEM.CSV", FILE_WRITE);
      if (f) {
        f.println("SampleNo,Ax_g,Ay_g,Az_g,TempIMU_C,Pressure_Pa,TempBaro_C");
        f.close();
        Serial.println("Created TELEM.CSV with header.");
      }
    } else {
      Serial.println("Appending to existing TELEM.CSV.");
    }
  }

  if (!sdReady || !mpuReady || !bmpReady) {
    Serial.println("WARNING: One or more peripherals failed to initialise.");
    Serial.println("Logger will only log from available sensors.");
  }

  Serial.println("Logging at 2 Hz. Ctrl+C to stop.");
  Serial.println("SampleNo,Ax_g,Ay_g,Az_g,TempIMU_C,Pressure_Pa,TempBaro_C");
}

void loop() {
  if (!sdReady) {
    delay(2000);
    return;
  }

  nowMs = millis();
  if (nowMs - lastLogMs < 500) return;
  lastLogMs = nowMs;

  sampleNo++;

  // ======== Read MPU-6050 Accelerometer ========
  axRaw = 0; ayRaw = 0; azRaw = 0; mpuTempRaw = 0;
  axG   = 0.0; ayG = 0.0; azG = 0.0; mpuTempC = 0.0;

  if (mpuReady) {
    // Accel (6 bytes)
    Wire.beginTransmission(MPU_ADDR);
    Wire.write(MPU_ACCEL_XOUT);
    Wire.endTransmission(false);
    Wire.requestFrom(MPU_ADDR, 6);

    if (Wire.available() >= 6) {
      rawMsb = Wire.read(); rawLsb = Wire.read();
      axRaw  = (int16_t)((rawMsb << 8) | rawLsb);
      rawMsb = Wire.read(); rawLsb = Wire.read();
      ayRaw  = (int16_t)((rawMsb << 8) | rawLsb);
      rawMsb = Wire.read(); rawLsb = Wire.read();
      azRaw  = (int16_t)((rawMsb << 8) | rawLsb);

      axG = (float)axRaw / 16384.0;
      ayG = (float)ayRaw / 16384.0;
      azG = (float)azRaw / 16384.0;
    }

    // Temperature (2 bytes at 0x41)
    Wire.beginTransmission(MPU_ADDR);
    Wire.write(MPU_TEMP_OUT);
    Wire.endTransmission(false);
    Wire.requestFrom(MPU_ADDR, 2);

    if (Wire.available() >= 2) {
      rawMsb     = Wire.read();
      rawLsb     = Wire.read();
      mpuTempRaw = (int16_t)((rawMsb << 8) | rawLsb);
      // MPU-6050 temp formula: Temp_C = RawTemp / 340.0 + 36.53
      mpuTempC   = (float)mpuTempRaw / 340.0 + 36.53;
    }
  }

  // ======== Read BMP280 ========
  bmpPressurePa = 0.0;
  bmpTempC      = 0.0;

  if (bmpReady) {
    bmpTempC      = bmp.readTemperature();   // degrees Celsius
    bmpPressurePa = bmp.readPressure();      // Pascals
  }

  // ======== Assemble CSV Line ========
  csvLine  = String(sampleNo)      + ",";
  csvLine += String(axG, 4)        + ",";
  csvLine += String(ayG, 4)        + ",";
  csvLine += String(azG, 4)        + ",";
  csvLine += String(mpuTempC, 2)   + ",";
  csvLine += String(bmpPressurePa, 1) + ",";
  csvLine += String(bmpTempC, 2);

  // ======== Write to SD Card ========
  File logFile = SD.open("TELEM.CSV", FILE_WRITE);
  if (logFile) {
    logFile.println(csvLine);
    logFile.close();
    Serial.println(csvLine);
  } else {
    Serial.print("ERROR: Cannot write sample #");
    Serial.println(sampleNo);
  }
}
```

## What to Click in MbedO

1. Drag **VEGA ARIES v3**, **MPU-6050**, **BMP280**, and **SD Card** components onto the canvas.
2. Wire the MPU-6050 and BMP280: **SDA → GPIO 0**, **SCL → GPIO 1**, **VCC → 3V3**, **GND → GND**. Connect **MPU-6050 AD0 → GND** and **BMP280 SDO → GND**.
3. Wire the SD Card: **SCK → GPIO 6**, **MOSI → GPIO 7**, **MISO → GPIO 4**, **CS → GPIO 9**, **VCC → 3V3**, **GND → GND**.
4. Paste the code into the editor and select **Interpreted Mode**.
5. Click **Run**.
6. Adjust the MPU-6050 widget's acceleration sliders and the BMP280 widget's pressure/temperature sliders to simulate sensor readings.
7. Observe the Serial Monitor and open `TELEM.CSV` in the SD card file browser to verify logged rows.

## Expected Output

Serial Monitor:
```
=== Dual-protocol Telemetry Logger ===
I2C sensors: MPU-6050 + BMP280
SPI storage: SD Card -> TELEM.CSV
-------------------------------------------
MPU-6050 ready at 0x68.
BMP280 ready at 0x76.
SD card ready.
Created TELEM.CSV with header.
Logging at 2 Hz. Ctrl+C to stop.
SampleNo,Ax_g,Ay_g,Az_g,TempIMU_C,Pressure_Pa,TempBaro_C
1,0.0011,-0.0023,1.0001,36.76,101325.0,24.50
2,0.0010,-0.0022,1.0002,36.77,101320.5,24.51
3,0.1250, 0.0010,0.9923,36.76,101310.0,24.52
```

`TELEM.CSV` on SD card:
```
SampleNo,Ax_g,Ay_g,Az_g,TempIMU_C,Pressure_Pa,TempBaro_C
1,0.0011,-0.0023,1.0001,36.76,101325.0,24.50
2,0.0010,-0.0022,1.0002,36.77,101320.5,24.51
3,0.1250,0.0010,0.9923,36.76,101310.0,24.52
```

## Expected Canvas Behavior

* Telemetry rows are logged to `TELEM.CSV` at 2 Hz (every 500 ms) and simultaneously printed to the Serial Monitor.
* Adjusting MPU-6050 widget acceleration sliders changes the Ax/Ay/Az columns in the next row.
* Adjusting the BMP280 widget changes the Pressure_Pa and TempBaro_C columns.
* If only one sensor is available, the other sensor's columns are filled with `0.00` and logging continues uninterrupted.
* If the SD card fails, the loop prints an error every 500 ms and does not crash.

## Code Walkthrough

| Line | Check / Action |
| --- | --- |
| `Wire.begin()` + `SPI.begin()` | Both buses are initialised in `setup()` — they operate independently and concurrently. |
| `Wire.write(0x00)` to `MPU_PWR_MGMT_1` | Clears the sleep bit so the MPU-6050 begins taking measurements. |
| `bmp.begin(0x76)` | Attempts to find the BMP280 at I2C address 0x76 and loads factory calibration coefficients. |
| `bmp.setSampling(...)` | Configures BMP280 oversampling, IIR filter, and standby time for stable pressure readings. |
| `Wire.requestFrom(MPU_ADDR, 6)` | Requests 6 consecutive bytes starting from `ACCEL_XOUT` for Ax, Ay, Az (MSB+LSB each). |
| `(int16_t)((rawMsb << 8) \| rawLsb)` | Reassembles the 16-bit signed integer from two bytes in big-endian order. |
| `mpuTempC = mpuTempRaw / 340.0 + 36.53` | Applies the MPU-6050 datasheet formula to convert raw ADC counts to °C. |
| `bmp.readPressure()` | Returns calibrated pressure in Pascals using the BMP280's embedded compensation algorithm. |
| `nowMs - lastLogMs < 500` | Non-blocking 500 ms timer — prevents SD writes from blocking the loop. |
| `SD.open("TELEM.CSV", FILE_WRITE)` | Opens file in append mode; closing after each write ensures every record is flushed to FAT. |

## Hardware & Safety Concept

* **I2C + SPI Simultaneously**: The ARIES v3 has separate I2C0 and SPI0 peripheral controllers sharing no pins. The firmware reads all I2C sensors first (blocking Wire transactions, typically <1 ms each), then opens the SD card (SPI transaction, typically 5–20 ms for a small write). These are sequential in `loop()` but the two buses are electrically independent — activating SPI CS does not affect I2C signals, and vice versa.
* **BMP280 Calibration**: The BMP280 stores factory-trimmed compensation coefficients in its internal ROM. The `Adafruit_BMP280` library reads these coefficients at `begin()` and applies them in `readPressure()` and `readTemperature()`. Without compensation, the raw ADC values have errors of many hundred Pascals. Never use raw ADC reads from BMP280 without the compensation polynomial.
* **SD Write Frequency vs. Wear**: At 2 Hz with ~60-byte rows, `TELEM.CSV` grows by approximately 120 bytes/second or ~432 MB/hour at sustained logging. For deployments longer than a few minutes, log at 0.1–0.5 Hz to extend SD card write endurance (typically 100,000 erase cycles per block). The non-blocking timer pattern in this firmware makes changing the log rate as simple as updating the `500` millisecond threshold.
* **Data Integrity on Power Loss**: The code closes the file after every write (`logFile.close()`). This flushes the FAT directory entry and ensures every committed row is recoverable even if the board loses power immediately after the close. Leaving a file open and relying on a final `close()` at shutdown risks losing the entire session if power fails unexpectedly.

## Try This! (Challenges)

1. **Altitude from Pressure**: Add a global `float altitudeM = 0.0` variable. After reading `bmpPressurePa`, compute `altitudeM = bmp.readAltitude(1013.25)` (sea-level reference pressure in hPa) and append it as an extra column to the CSV row. Open the file in a spreadsheet to plot altitude vs. sample number.
2. **Session Statistics**: Add global float variables `minTempC`, `maxTempC`, `minPressure`, `maxPressure`, all initialised to the first valid reading. After each sample, update them with the new reading if it is outside the current range. Print the session min/max report to the Serial Monitor every 10 samples using a global `int reportCounter` that resets after printing.

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| `ERROR: MPU-6050 not found. I2C error code: 2` | AD0 not tied to GND, or SDA/SCL swapped | Confirm GPIO 0 = SDA, GPIO 1 = SCL; confirm AD0 is connected to GND. |
| `ERROR: BMP280 not found at 0x76` | SDO pin floating or module at 0x77 | Tie SDO to GND for 0x76; change `bmp.begin(0x77)` if SDO is tied to VCC. |
| `ERROR: SD card initialisation failed` | CS pin wrong or card not FAT32 | Confirm CS is GPIO 9; reformat card as FAT32. |
| `TempIMU_C` reads ~36–40 °C even at room temperature | Expected — MPU-6050 reads its own die temperature, not ambient | Use BMP280's `TempBaro_C` for ambient temperature. |
| Pressure reads 0.0 Pa | BMP280 in forced mode not triggered | `setSampling(MODE_NORMAL, ...)` should auto-sample; confirm library version supports this. |
| CSV rows appear but all zeros | Sensor reads returning early due to `Wire.available() < 6` | Check pull-up resistors on SDA/SCL; 4.7 kΩ to 3V3 required for reliable I2C. |

## Mode Notes

This project runs in MbedO **interpreted C++ mode**.

## Related Projects

- [144 - RFID Access Logger](144-rfid-access-logger.md)
- [147 - GPS Velocity Logger](147-gps-velocity-logger.md)
- [149 - I2C OLED Tilt-compensated Compass](149-i2c-oled-tilt-compensated-compass.md)
