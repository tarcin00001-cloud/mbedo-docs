# 78 - Pico BMP180 Serial

Read and log atmospheric pressure and temperature data from a BMP180 barometric sensor to the Serial Monitor.

## Goal
Learn how to interface I2C pressure sensors, read compensation parameters, and print calibrated telemetry.

## What You Will Build
An atmospheric monitor:
- **BMP180 Sensor (GP4 SDA, GP5 SCL)**: Reads pressure (Pa), temperature (°C), and calculates relative altitude (meters), logging them to the Serial Monitor every 1.5 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| BMP180 Barometric Sensor | `bmp180` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| BMP180 Sensor | VCC | 3.3V | Power supply |
| BMP180 Sensor | SDA | GP4 | I2C Data |
| BMP180 Sensor | SCL | GP5 | I2C Clock |
| BMP180 Sensor | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_BMP085.h>

Adafruit_BMP085 bmp;

void setup() {
  Serial.begin(9600);
  
  // Set Pico I2C pins
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  if (!bmp.begin()) {
    Serial.println("Error: Could not find a valid BMP085/BMP180 sensor!");
  } else {
    Serial.println("BMP180 Sensor Online");
  }
}

void loop() {
  float temp = bmp.readTemperature();
  long pressure = bmp.readPressure();
  float altitude = bmp.readAltitude();

  Serial.print("Temp: ");
  Serial.print(temp);
  Serial.print(" C | Pressure: ");
  Serial.print(pressure);
  Serial.print(" Pa | Altitude: ");
  Serial.print(altitude);
  Serial.println(" m");

  delay(1500);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **BMP180 Sensor** onto the canvas.
2. Connect BMP180: **VCC** to **3.3V**, **SDA** to **GP4**, **SCL** to **GP5**, **GND** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the pressure slider on the BMP180 model and view the Serial logs.

## Expected Output

Terminal:
```
BMP180 Sensor Online
Temp: 24.50 C | Pressure: 101325 Pa | Altitude: 112.50 m
```

## Expected Canvas Behavior
| BMP180 Slider Value | Serial Pressure | Serial Altitude |
| --- | --- | --- |
| High Pressure | ~102500 Pa | Lower altitude |
| Standard (Sea Level)| ~101325 Pa | ~0 m |
| Low Pressure | ~95000 Pa | Higher altitude |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `bmp.readPressure()` | Reads internal raw pressure counts and applies factory calibration parameters to return the pressure in Pascals. |
| `bmp.readAltitude()` | Calculates relative altitude based on standard sea-level pressure values. |

## Hardware & Safety Concept: Barometric Altitude Calculations
Barometric sensors measure the absolute pressure of the air. Because air density decreases as you go higher, altitude can be calculated using the barometric formula. However, local weather fronts (high and low pressure zones) also change atmospheric pressure. To get an accurate absolute altitude, you must feed the current sea-level pressure of your local airport into the library function (`bmp.readAltitude(101325)`).

## Try This! (Challenges)
1. **Pressure Unit Scale**: Convert the pressure reading from Pascals to millibars (1 hPa = 100 Pa) or inches of Mercury (inHg) and print it.
2. **Altitude Alert**: Light an LED on GP15 if the calculated altitude changes by more than 10 meters from startup.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Serial displays "Could not find a valid BMP085/BMP180 sensor!" | I2C address conflict | Verify the wiring connections to GP4 and GP5. Ensure the SDA and SCL lines are not swapped. |

## Mode Notes
This basic I2C sensor project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [71 - Pico DHT Serial](71-pico-dht-serial.md)
- [79 - Pico BMP180 LCD](79-pico-bmp180-lcd.md)
