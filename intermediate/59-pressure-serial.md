# 59 - Pressure Serial

Measure and print barometric pressure in Pascals using a BMP180 sensor.

## Goal
Learn how to use the I2C bus to interface with a barometric pressure sensor (BMP180) and print raw pressure readouts.

## What You Will Build
The Arduino reads atmospheric pressure from the BMP180 sensor via the I2C bus and prints the value in Pascals (Pa) and hectopascals (hPa) to the Terminal every 1 second.

**Why A4 and A5?** Pins A4 (SDA) and A5 (SCL) comprise the I2C bus required to request digital data from the barometric sensor chip.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| BMP180 Sensor | `bmp180` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| BMP180 Sensor | VCC | 3.3V | Power supply (3.3V or 5V depending on breakout) |
| BMP180 Sensor | SDA | A4 | I2C Serial Data connection |
| BMP180 Sensor | SCL | A5 | I2C Serial Clock connection |
| BMP180 Sensor | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_BMP085.h> // Library shim supports BMP180/BMP085

Adafruit_BMP085 bmp;

void setup() {
  Serial.begin(9600);
  Serial.println("BMP180 Barometer Monitor Ready");
  
  // Initialize the sensor (defaults to 0x77 address)
  if (!bmp.begin()) {
    Serial.println("Error: Could not find a valid BMP180 sensor!");
    while (1) {} // Halt program
  }
}

void loop() {
  // Read atmospheric pressure in Pascals
  int32_t pressurePa = bmp.readPressure();
  
  // Convert to hectopascals (1 hPa = 100 Pa)
  float pressureHPa = pressurePa / 100.0;
  
  Serial.print("Pressure: ");
  Serial.print(pressurePa);
  Serial.print(" Pa | ");
  Serial.print(pressureHPa);
  Serial.println(" hPa");
  
  delay(1000); // Poll every second
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** and **BMP180 Sensor** onto the canvas.
2. Connect BMP180 **VCC** to Arduino **3.3V** (or **5V**), **GND** to Arduino **GND**, **SDA** to Arduino **A4**, and **SCL** to Arduino **A5**.
3. Paste the code into the editor.
4. Use the default Arduino interpreted runtime.
5. Click **Run**.
6. Open the **Terminal** tab in the bottom dock.
7. Double-click the BMP180 sensor, adjust the pressure slider, and watch the values change in the Terminal.

## Expected Output

Terminal:
```
BMP180 Barometer Monitor Ready
Pressure: 101325 Pa | 1013.25 hPa
Pressure: 100850 Pa | 1008.50 hPa
...
```

### Expected Canvas Behavior

| Slider Pressure Input | Pin A4/A5 Activity | Terminal Output |
| --- | --- | --- |
| 1013 hPa (Standard) | I2C transmissions | "Pressure: 101300 Pa | 1013.00 hPa" |
| 985 hPa (Low) | I2C transmissions | "Pressure: 98500 Pa | 985.00 hPa" |

The Terminal updates to match the sensor slider value in real-time.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `#include <Adafruit_BMP085.h>` | Imports the sensor library. The BMP180 is register-compatible with the older BMP085, allowing them to share this driver. |
| `bmp.begin()` | Starts I2C communications and reads the factory calibration coefficients stored in the sensor's EEPROM. |
| `bmp.readPressure()` | Performs raw pressure reading and compensates it using the calibration data, returning an integer in Pascals. |

## Hardware & Safety Concept: Barometric Pressure
Atmospheric pressure is the force exerted by the weight of air molecules in the atmosphere above a surface.
- At sea level, standard atmospheric pressure is roughly **101,325 Pa** (1013.25 hPa or 1 atm).
- As you go up in altitude, there are fewer air molecules above you, so pressure decreases.
- High-pressure systems generally bring clear, dry weather, whereas low-pressure systems suggest oncoming storms or rain.

## Try This! (Challenges)
1. **Weather Predictor**: Add logic to print a warning message "Storm Warning" if the pressure drops below `1000.0` hPa.
2. **Standard Atmosphere check**: Calculate and print the difference between the current reading and standard sea-level pressure (1013.25 hPa).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Prints "Error: Could not find..." | I2C wiring swapped | Ensure SDA goes to A4 and SCL goes to A5. If reversed, the Arduino cannot locate the sensor address. |
| Value stays constant regardless of slider | Reading fail or code loop lock | Verify that the initialization check `bmp.begin()` returned true. |

## Mode Notes
The `Adafruit_BMP085` / `BMP180` library is fully shimmed in the MbedO interpreted mode runtime.

## Related Projects
- [15 - Analog Meter Serial](../beginner/15-analog-meter-serial.md)
- [60 - BMP180 Temperature Serial](60-bmp-180-temperature-serial.md)
- [61 - Weather Readout LCD](61-weather-readout-lcd.md)
