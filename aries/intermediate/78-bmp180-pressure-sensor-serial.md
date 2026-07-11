# 78 - BMP180 Pressure Sensor Serial Logs

Read raw atmospheric pressure and temperature data from a BMP180 sensor and print the log metrics to the Serial Monitor.

## Goal
Learn how to interface I2C barometric pressure sensors, initialize the BMP180 device, and read pressure values in hectopascals (hPa).

## What You Will Build
A BMP180 sensor is connected to the hardware I2C0 interface of the ARIES v3 board. The board queries the sensor every 2 seconds and logs the barometric pressure and ambient temperature to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| BMP180 Barometric Pressure Sensor | `bmp180` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| BMP180 Sensor | VCC | 3V3 | Red | Power input (3.3V) |
| BMP180 Sensor | GND | GND | Black | Ground reference |
| BMP180 Sensor | SDA | SDA0 (GP17) | Blue | I2C Data line |
| BMP180 Sensor | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** The BMP180 sensor is an I2C peripheral. Ensure SDA connects to SDA0 (GP17) and SCL connects to SCL0 (GP16). Power the sensor from 3.3V, not 5V.

## Code
```cpp
// BMP180 Pressure Sensor Serial Logs
#include <Wire.h>
#include <Adafruit_BMP085.h>

Adafruit_BMP085 bmp;
bool sensorOk = true;

void setup() {
  Serial.begin(115200);
  Wire.begin(); // Initialize default I2C0 interface
  
  if (!bmp.begin()) {
    Serial.println("Could not find a valid BMP180 sensor, check wiring!");
    sensorOk = false;
  } else {
    Serial.println("BMP180 Sensor initialized successfully.");
  }
}

void loop() {
  if (!sensorOk) {
    delay(2000);
    return; // Stop execution loop if sensor fails
  }
  
  // Read atmospheric pressure in Pascals and convert to hPa (1 hPa = 100 Pa)
  float pressure = bmp.readPressure() / 100.0;
  float temperature = bmp.readTemperature(); // Celsius
  
  Serial.print("Pressure: ");
  Serial.print(pressure, 1);
  Serial.print(" hPa  |  Temperature: ");
  Serial.print(temperature, 1);
  Serial.println(" *C");
  
  delay(2000); // 0.5Hz refresh rate
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** and **BMP180 Sensor** components onto the canvas.
2. Wire the sensor: **VCC** to **3V3**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.
6. Adjust the pressure slider on the BMP180 widget and observe changes in the Serial Monitor.

## Expected Output
Serial Monitor:
```
BMP180 Sensor initialized successfully.
Pressure: 1013.2 hPa  |  Temperature: 25.4 *C
Pressure: 1008.5 hPa  |  Temperature: 25.5 *C
```

## Expected Canvas Behavior
* At startup, the Serial Monitor verifies sensor connection.
* Updated pressure and temperature logs print to the Serial Monitor every 2 seconds.
* Adjusting the slider on the BMP180 component updates the logged pressure instantly on the next loop cycle.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `#include <Adafruit_BMP085.h>` | Imports the Adafruit BMP085/BMP180 library. |
| `bmp.begin()` | Initialises calibration registers on the BMP180 chip. |
| `bmp.readPressure()` | Reads raw pressure value in Pascals (Pa). |
| `pressure / 100.0` | Converts raw Pascal readings into standard meteorology hectopascals (hPa). |

## Hardware & Safety Concept
* **Barometric Pressure Sensing**: The BMP180 uses a piezo-resistive sensor element to measure atmospheric pressure. Inside the chip is a flexible silicon diaphragm that deflects under changes in gas pressure. This deformation changes its electrical resistance, which is measured by a high-resolution 16-to-19 bit analog-to-digital converter (ADC) on the chip and sent over I2C.

## Try This! (Challenges)
1. **Sea Level Pressure**: Read and print the estimated pressure at sea level using `bmp.readSealevelPressure(altitude_meters)`.
2. **Altitude Tracker**: Print estimated altitude relative to standard sea level pressure using the `bmp.readAltitude()` function.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| "Could not find a valid BMP180..." | Wrong pins or wiring reversed | Verify SDA connects to GP17, SCL connects to GP16. |
| Pressure readings are static | I2C lockup | Restart the simulation to re-initialise the I2C bus. |
| Temperature is inaccurate | Internal self-heating | Ensure the sensor is placed away from heat-generating components. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [79 - BMP180 Temp & Altitude Display](79-bmp180-temp-altitude-display.md)
- [80 - BMP180 Weather Trend LCD](80-bmp180-weather-trend-lcd.md)
- [71 - DHT22 Temp & Humidity Serial Logs](71-dht22-temp-humidity-serial.md)
