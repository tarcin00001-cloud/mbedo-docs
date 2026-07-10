# 78 - ESP32 BMP180 Pressure Sensor Serial Logs

Read barometric pressure and temperature from a BMP180 sensor and output formatted measurement logs to the Serial Monitor.

## Goal
Learn how to interface the BMP180 pressure sensor using I2C communication, initialises the device using the Adafruit BMP085/BMP180 library, and read pressure in Pascals (Pa) and hectopascals (hPa).

## What You Will Build
A BMP180 pressure sensor connected to the ESP32's I2C bus (SDA: GPIO 21, SCL: GPIO 22). The ESP32 queries the sensor every second, reads barometric pressure and temperature, and prints formatted logs to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| BMP180 Barometric Pressure Sensor | `bmp180` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| BMP180 Sensor | VCC | 3V3 | Red | Sensor operates on 3.3V |
| BMP180 Sensor | GND | GND | Black | Ground reference |
| BMP180 Sensor | SDA | GPIO21 | Blue | I2C Data line |
| BMP180 Sensor | SCL | GPIO22 | Yellow | I2C Clock line |

> **Wiring tip:** The BMP180 is an I2C sensor. Connect SDA to GPIO 21 and SCL to GPIO 22. Most break-out boards include an onboard 3.3V voltage regulator and pull-up resistors on the I2C lines.

## Code
```cpp
// BMP180 Barometric Pressure Sensor Serial Logs
#include <Wire.h>
#include <Adafruit_BMP085.h> // Supports both BMP085 and BMP180

Adafruit_BMP085 bmp;

void setup() {
  Serial.begin(115200);
  Serial.println("Initialising BMP180 sensor...");
  
  if (!bmp.begin()) {
    Serial.println("Could not find a valid BMP180 sensor, check wiring!");
    while (1) {}
  }
  
  Serial.println("BMP180 ready.");
}

void loop() {
  float temperature = bmp.readTemperature();
  int32_t pressurePa = bmp.readPressure(); // Pressure in Pascals
  float pressureHpa = pressurePa / 100.0;  // 1 hPa = 100 Pa
  
  Serial.print("Temperature: ");
  Serial.print(temperature, 1);
  Serial.print(" °C  |  ");
  
  Serial.print("Pressure: ");
  Serial.print(pressurePa);
  Serial.print(" Pa (");
  Serial.print(pressureHpa, 2);
  Serial.println(" hPa)");
  
  delay(1000); // Poll once per second
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **BMP180 Sensor** onto the canvas.
2. Connect BMP180 **SDA** to **GPIO21** and **SCL** to **GPIO22**.
3. Paste the code and click **Run**.
4. Adjust the pressure slider on the BMP180 widget and observe the updated values in the Serial Monitor.

## Expected Output
Serial Monitor:
```
Initialising BMP180 sensor...
BMP180 ready.
Temperature: 24.5 °C  |  Pressure: 101325 Pa (1013.25 hPa)
Temperature: 24.5 °C  |  Pressure: 100800 Pa (1008.00 hPa)
```

## Expected Canvas Behavior
* The Serial Monitor prints updated sensor logs every second.
* Moving the pressure and temperature sliders on the BMP180 component updates the output instantly on the next loop cycle.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `#include <Adafruit_BMP085.h>` | Imports the Adafruit library that handles register communication with the BMP085/BMP180. |
| `bmp.begin()` | Verifies communication with the sensor over I2C (address 0x77). |
| `bmp.readTemperature()` | Reads the temperature from the sensor in Celsius. |
| `bmp.readPressure()` | Reads the raw barometric pressure in Pascals. |

## Hardware & Safety Concept: Barometric Pressure Sensing
The BMP180 is a high-precision, piezoresistive barometric pressure sensor. It contains a MEMS (Micro-Electro-Mechanical Systems) diaphragm that bends slightly under atmospheric pressure, changing its internal resistance. An onboard ADC reads this resistance and converts it to a digital pressure value. This data is used in weather monitoring stations, altimeters, and drone altitude stabilization.

## Try This! (Challenges)
1. **Fahrenheit Conversion**: Modify the code to display the temperature in Fahrenheit.
2. **Sea Level Pressure**: Read sea level pressure using `bmp.readSealevelPressure(altitude_meters)` to calculate pressure adjusted for your location.
3. **Pressure Trend Alert**: Save the pressure reading every 10 seconds. Print a warning if the pressure drops rapidly (indicating an approaching storm).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| "Could not find a valid BMP180..." | Sensor not powered or I2C lines swapped | Verify VCC connects to 3.3V; check SDA (GPIO 21) and SCL (GPIO 22) connections |
| Temperature reads extremely high | Internal self-heating | Avoid polling the sensor too rapidly (e.g. less than 100ms intervals) |
| Constant pressure readings | Slider not adjusting simulated input | Double-check sensor node address configurations |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [79 - ESP32 BMP180 Temp & Altitude Display](79-esp32-bmp180-temp-altitude-display.md)
- [80 - ESP32 BMP180 Weather Trend LCD](80-esp32-bmp180-weather-trend-lcd.md)
- [71 - ESP32 DHT22 Temp & Humidity Serial Logs](71-esp32-dht22-temp-humidity-serial-logs.md)
