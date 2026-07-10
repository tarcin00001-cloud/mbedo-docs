# 60 - BMP180 Temperature Serial

Measure and print ambient temperature in Celsius using a BMP180 barometric sensor.

## Goal
Learn how to read temperature values from a BMP180 sensor's internal thermal sensor and print them to the Terminal.

## What You Will Build
The Arduino reads ambient temperature from the BMP180 sensor over I2C and prints the temperature (in Celsius) to the Terminal every 1 second.

**Why A4 and A5?** Pins A4 and A5 comprise the I2C bus used to query the sensor registers.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| BMP180 Sensor | `bmp180` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| BMP180 Sensor | VCC | 3.3V | Power supply |
| BMP180 Sensor | SDA | A4 | I2C Serial Data line |
| BMP180 Sensor | SCL | A5 | I2C Serial Clock line |
| BMP180 Sensor | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_BMP085.h>

Adafruit_BMP085 bmp;

void setup() {
  Serial.begin(9600);
  Serial.println("BMP180 Thermometer Ready");
  
  if (!bmp.begin()) {
    Serial.println("Error: BMP180 initialization failed!");
    while (1) {}
  }
}

void loop() {
  // Read temperature as a float in Celsius
  float tempC = bmp.readTemperature();
  
  Serial.print("Temperature: ");
  Serial.print(tempC);
  Serial.println(" C");
  
  delay(1000);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** and **BMP180 Sensor** onto the canvas.
2. Connect BMP180 **VCC** to Arduino **3.3V**, **GND** to Arduino **GND**, **SDA** to Arduino **A4**, and **SCL** to Arduino **A5**.
3. Paste the code into the editor.
4. Use the default Arduino interpreted runtime.
5. Click **Run**.
6. Open the **Terminal** tab in the bottom dock.
7. Double-click the BMP180 sensor on the canvas, adjust the temperature slider, and watch the values print in the Terminal.

## Expected Output

Terminal:
```
BMP180 Thermometer Ready
Temperature: 23.50 C
Temperature: 28.10 C
...
```

### Expected Canvas Behavior

| Slider Temperature Input | Pin A4/A5 Activity | Terminal Output |
| --- | --- | --- |
| 20C | I2C communication active | "Temperature: 20.00 C" |
| 35C | I2C communication active | "Temperature: 35.00 C" |

The Terminal updates to match the sensor properties in real-time.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `bmp.readTemperature()` | Reads the temperature value in Celsius. The barometer chip measures temperature to perform internal pressure calculations, but makes this sensor data available to the user. |

## Hardware & Safety Concept: Sensor Temperature Compensation
Silicon pressure sensor diaphragms expand and contract with temperature changes, altering their electrical properties. To ensure accurate pressure readings, the BMP180 contains a built-in temperature sensor. Every time a pressure reading is taken, the driver first reads the temperature and uses it to perform mathematical **temperature compensation** on the pressure raw values.

## Try This! (Challenges)
1. **Fahrenheit Conversion**: Modify the code to calculate and display the temperature in Fahrenheit.
2. **Thermal Warning**: Add an LED on D13 and make it turn ON if the temperature exceeds 30C.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Prints "Error: BMP180 initialization failed!" | I2C pins swapped | Ensure SDA goes to A4 and SCL goes to A5. |
| Reading is constant and never changes | Calibration values failed to read | Verify the initialization succeeded. Reset the simulation and retry. |

## Mode Notes
These patterns (BMP180 I2C temperature read and printing) are supported by MbedO interpreted mode.

## Related Projects
- [20 - Thermistor Raw Print](../beginner/20-thermistor-raw-print.md)
- [59 - Pressure Serial](59-pressure-serial.md)
- [61 - Weather Readout LCD](61-weather-readout-lcd.md)
