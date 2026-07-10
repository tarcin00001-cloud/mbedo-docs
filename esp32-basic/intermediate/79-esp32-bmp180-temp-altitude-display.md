# 79 - ESP32 BMP180 Temp & Altitude Display

Display barometric temperature and calculated altitude from a BMP180 sensor on a 16x2 I2C LCD screen.

## Goal
Learn how to use barometric formula equations to calculate altitude from atmospheric pressure, and display the values on an I2C LCD screen.

## What You Will Build
A BMP180 sensor reads pressure and temperature, and a 16x2 I2C LCD displays the temperature in Celsius on Row 1 and the estimated altitude in meters on Row 2.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| BMP180 Barometric Pressure Sensor | `bmp180` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| BMP180 Sensor | VCC | 3V3 | Red | Power |
| BMP180 Sensor | GND | GND | Black | Ground |
| BMP180 Sensor | SDA | GPIO21 | Blue | I2C Data (shared) |
| BMP180 Sensor | SCL | GPIO22 | Yellow | I2C Clock (shared) |
| I2C LCD | VCC | 5V (Vin) | Red | LCD Power |
| I2C LCD | GND | GND | Black | Ground |
| I2C LCD | SDA | GPIO21 | Blue | I2C Data (shared) |
| I2C LCD | SCL | GPIO22 | Yellow | I2C Clock (shared) |

> **Wiring tip:** Connect both I2C devices (BMP180 and LCD) in parallel to GPIO 21 and 22. Each device has a unique I2C address, allowing the ESP32 to communicate with them separately on the same bus.

## Code
```cpp
// BMP180 Temp & Altitude Display
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Adafruit_BMP085.h>

Adafruit_BMP085 bmp;
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Standard atmospheric pressure at sea level is 1013.25 hPa (101325 Pa)
// Set this to your local pressure adjusted to sea level for accurate altitude
const float SEA_LEVEL_PRESSURE = 101325; 

void setup() {
  Serial.begin(115200);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Altimeter Setup");
  
  if (!bmp.begin()) {
    lcd.setCursor(0, 1);
    lcd.print("Sensor error!  ");
    while (1) {}
  }
  
  delay(1000);
  lcd.clear();
}

void loop() {
  float temp = bmp.readTemperature();
  // Read altitude based on standard sea level pressure reference
  float altitude = bmp.readAltitude(SEA_LEVEL_PRESSURE); 
  
  // Display Temp
  lcd.setCursor(0, 0);
  lcd.print("Temp: ");
  lcd.print(temp, 1);
  lcd.print((char)223); // Degree symbol
  lcd.print("C    ");
  
  // Display Altitude
  lcd.setCursor(0, 1);
  lcd.print("Altitude: ");
  lcd.print(altitude, 1);
  lcd.print("m   ");
  
  Serial.print("Temp: "); Serial.print(temp);
  Serial.print(" | Altitude: "); Serial.print(altitude); Serial.println(" m");
  
  delay(1000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **BMP180 Sensor**, and **16x2 I2C LCD** onto the canvas.
2. Wire SDA to **GPIO21** and SCL to **GPIO22** for both components.
3. Paste the code and click **Run**.
4. Adjust the pressure slider on the BMP180 widget and observe the calculated altitude change on the LCD.

## Expected Output
Serial Monitor:
```
Temp: 24.5 | Altitude: 0.0 m
Temp: 24.5 | Altitude: 44.2 m
```

LCD Display (at standard pressure):
```
Temp: 24.5°C
Altitude: 0.0m
```

## Expected Canvas Behavior
* The LCD displays the temperature and altitude, refreshing every second.
* Increasing the simulated barometric pressure on the BMP180 widget decreases the calculated altitude. Decreasing pressure increases altitude.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `bmp.readAltitude(SEA_LEVEL_PRESSURE)` | Calculates altitude in meters using standard pressure equations. |
| `SEA_LEVEL_PRESSURE = 101325` | Defines the reference pressure at sea level in Pascals (Pa). |
| `lcd.print("m   ")` | Appends spaces to clear any old digits when the number of digits drops. |

## Hardware & Safety Concept: Barometric Altitude Calculation
The weight of the atmosphere decreases as we climb higher. This relationship is calculated using the barometric formula:
\[A = 44330 \times \left(1 - \left(\frac{P}{P_0}\right)^{\frac{1}{5.255}}\right)\]
where \(A\) is altitude in meters, \(P\) is current pressure, and \(P_0\) is sea level pressure. Because local weather conditions change atmospheric pressure dynamically, Altimeters must be periodically calibrated with a known reference pressure to prevent altitude drift.

## Try This! (Challenges)
1. **Fahrenheit/Feet Toggle**: Add a button on GPIO 4 that switches the unit displays from Celsius/meters to Fahrenheit/feet.
2. **Relative Altitude Zero**: Add a button on GPIO 15 that sets the current altitude as "0" (ground level reference) to measure relative height differences.
3. **High Altitude Warning**: Flash the LCD backlight if the altitude climbs above a threshold.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Altitude readings drift over time | Weather front changing local barometric pressure | Calibrate by updating the `SEA_LEVEL_PRESSURE` variable to match local airport reports |
| LCD display shows constant value | BMP180 not updating | Confirm the sensor VCC pin is connected to 3.3V, not GND |
| Negative altitude readings | Local pressure is higher than standard sea level | Normal behavior under high-pressure weather systems |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [78 - ESP32 BMP180 Pressure Sensor Serial Logs](78-esp32-bmp180-pressure-sensor-serial-logs.md)
- [80 - ESP32 BMP180 Weather Trend LCD](80-esp32-bmp180-weather-trend-lcd.md)
- [58 - ESP32 16×2 I2C LCD Print Text](58-esp32-16x2-i2c-lcd-print-text.md)
