# 62 - Altitude Print

Calculate and print your current altitude in meters using barometric pressure readings.

## Goal
Learn how to use a barometric sensor (BMP180) to estimate altitude changes based on atmospheric pressure drops.

## What You Will Build
The Arduino reads atmospheric pressure, calculates the corresponding altitude in meters above sea level using the built-in library function, and prints it to the Terminal every 1 second.

**Why A4 and A5?** Pins A4 and A5 comprise the I2C interface required to poll the sensor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| BMP180 Sensor | `bmp180` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| BMP180 Sensor | VCC | 3.3V | Power supply |
| BMP180 Sensor | SDA | A4 | I2C Serial Data connection |
| BMP180 Sensor | SCL | A5 | I2C Serial Clock connection |
| BMP180 Sensor | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_BMP085.h>

Adafruit_BMP085 bmp;

// Standard sea-level atmospheric pressure in Pascals (1013.25 hPa)
const float SEA_LEVEL_PRESSURE = 101325.0;

void setup() {
  Serial.begin(9600);
  Serial.println("BMP180 Altimeter Ready");
  
  if (!bmp.begin()) {
    Serial.println("Error: BMP180 initialization failed!");
    while (1) {}
  }
}

void loop() {
  // Read calculated altitude in meters above sea level
  float altitude = bmp.readAltitude(SEA_LEVEL_PRESSURE);
  float pressure = bmp.readPressure() / 100.0;
  
  Serial.print("Pressure: ");
  Serial.print(pressure);
  Serial.print(" hPa | Calculated Altitude: ");
  Serial.print(altitude);
  Serial.println(" meters");
  
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
7. Double-click the BMP180 sensor on the canvas, adjust the pressure slider down (simulating going up in altitude), and watch the calculated altitude rise in the Terminal.

## Expected Output

Terminal:
```
BMP180 Altimeter Ready
Pressure: 1013.25 hPa | Calculated Altitude: 0.00 meters
Pressure: 890.50 hPa | Calculated Altitude: 1085.30 meters
...
```

### Expected Canvas Behavior

| Pressure Input | Calculated Altitude | Altimeter Action |
| --- | --- | --- |
| 1013 hPa | 0.0 meters | Sea Level Reference |
| 850 hPa | ~1450 meters | Climbing (Altitude rises) |

The calculated altitude climbs as the pressure slider decreases.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `bmp.readAltitude(SEA_LEVEL_PRESSURE)` | Calculates altitude based on standard sea-level pressure. It compares the current pressure reading against the reference value to estimate elevation. |

## Hardware & Safety Concept: Barometric Altimeters
Barometric altimeters calculate height using the **barometric formula**. Since air pressure is caused by the weight of air above, pressure decreases as you climb. However, weather fronts also change local pressure.
- If a low-pressure storm front moves in, a barometric altimeter will perceive this as a climb in altitude, even if you remain stationary.
- For accurate readings in aviation or hiking, altimeters must be calibrated daily against a known local benchmark pressure.

## Try This! (Challenges)
1. **Feet Conversion**: Convert the output so that it displays the altitude in feet instead of meters. (Hint: multiply meters by `3.28084`).
2. **Relative Altitude**: Modify the code to record the initial pressure at startup and use that value as the baseline to calculate relative height changes (e.g. tracking height when walking up stairs).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Altitude shows negative numbers | Current pressure exceeds sea-level reference | This is normal if local atmospheric pressure is high. Update `SEA_LEVEL_PRESSURE` to match the local weather forecast. |
| Value stays stuck | Sensor failed initialization | Ensure wires are connected and reset the simulation. |

## Mode Notes
These patterns (BMP180 calculations and float printouts) are supported by MbedO interpreted mode.

## Related Projects
- [59 - Pressure Serial](59-pressure-serial.md)
- [60 - BMP180 Temperature Serial](60-bmp-180-temperature-serial.md)
