# 79 - BMP180 Temp & Altitude Display

Display ambient temperature and estimated altitude calculations from a BMP180 sensor onto a 16x2 I2C LCD.

## Goal
Learn how to run multiple I2C slave devices in parallel on the same bus, query sensor registers, and format outputs on a character display.

## What You Will Build
A BMP180 barometric sensor and an I2C 16x2 LCD are connected to the hardware I2C0 pins of the ARIES v3 board. The LCD shows the live temperature in Celsius on Row 1 and the calculated altitude in meters on Row 2.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| BMP180 Barometric Pressure Sensor | `bmp180` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| BMP180 Sensor | VCC | 3V3 | Red | Power input (3.3V) |
| BMP180 Sensor | GND | GND | Black | Ground reference |
| BMP180 Sensor | SDA | SDA0 (GP17) | Blue | I2C Data line (shared) |
| BMP180 Sensor | SCL | SCL0 (GP16) | Yellow | I2C Clock line (shared) |
| I2C LCD | VCC | 5V | Red | LCD power supply |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | SDA0 (GP17) | Blue | I2C Data line (shared) |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | I2C Clock line (shared) |

> **Wiring tip:** Share the I2C0 bus lines SDA0 (GP17) and SCL0 (GP16) in parallel on the breadboard for both the BMP180 and the I2C LCD. Make sure the BMP180 VCC connects to 3.3V and the LCD VCC connects to 5V.

## Code
```cpp
// BMP180 Temp & Altitude Display
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Adafruit_BMP085.h>

Adafruit_BMP085 bmp;
LiquidCrystal_I2C lcd(0x27, 16, 2);
bool sensorOk = true;

void setup() {
  Wire.begin(); // Initialize default I2C0 interface
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Altitude HUD");
  
  if (!bmp.begin()) {
    lcd.setCursor(0, 1);
    lcd.print("Sensor error!  ");
    sensorOk = false;
  } else {
    lcd.setCursor(0, 1);
    lcd.print("Sensor Ready   ");
    delay(1000);
    lcd.clear();
  }
}

void loop() {
  if (!sensorOk) {
    delay(2000);
    return;
  }
  
  float temp = bmp.readTemperature();
  // Read altitude assuming standard baseline sea level pressure (101325 Pascals)
  float altitude = bmp.readAltitude(101325);
  
  // Display Temperature on Line 1
  lcd.setCursor(0, 0);
  lcd.print("Temp: ");
  lcd.print(temp, 1);
  lcd.print(" C      "); // Trailing spaces to overwrite old text
  
  // Display Altitude on Line 2
  lcd.setCursor(0, 1);
  lcd.print("Alt:  ");
  lcd.print(altitude, 1);
  lcd.print(" m      ");
  
  delay(1500); // 1.5-second refresh interval
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **BMP180 Sensor**, and **I2C LCD Display** components onto the canvas.
2. Connect both BMP180 SDA and LCD SDA to **SDA0 (GP17)**. Connect both SCL pins to **SCL0 (GP16)**.
3. Wire BMP180 VCC to **3V3** and LCD VCC to **5V**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.
7. Change the pressure slider on the BMP180 widget and observe the updated altitude calculations on the LCD display.

## Expected Output
LCD Screen:
```
Temp: 24.5 C
Alt:  112.4 m
```

## Expected Canvas Behavior
* Row 1 displays the current temperature in Celsius.
* Row 2 displays the calculated altitude in meters.
* Sliding the pressure slider down on the BMP180 widget increases the displayed altitude (lower pressure matches higher altitude).

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `bmp.readTemperature()` | Queries the BMP180 temperature register. |
| `bmp.readAltitude(101325)` | Computes altitude based on standard sea-level reference pressure. |
| `lcd.setCursor(0, 0)` | Moves the writing cursor to column 0, row 0. |
| `lcd.print("Alt:  ")` | Prints the altitude text label to the screen. |

## Hardware & Safety Concept
* **Barometric Altimetry**: Atmospheric pressure decreases exponentially as altitude increases. The relationship is defined by the barometric formula. Because weather systems also change local atmospheric pressure, barometric altimeters must be calibrated against a known local sea-level pressure baseline (QNH) to display true altitude, otherwise, atmospheric shifts will appear as altitude drift.

## Try This! (Challenges)
1. **Fahrenheit Scale Display**: Modify the code to display the temperature in Fahrenheit.
2. **Dynamic Calibration**: Add a push button to GPIO 16. When pressed, take the current pressure reading and set it as the new baseline sea level reference pressure.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Display shows "Sensor error!" | Bad I2C wiring to BMP180 | Verify SDA is connected to GP17 and SCL to GP16. |
| LCD display does not light up | No 5V power | Make sure the LCD's VCC pin connects to the 5V rail on ARIES. |
| Altitude readings drift | Local weather front shifts | Calibrate the reference sea-level pressure in the `readAltitude()` parameter. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [78 - BMP180 Pressure Sensor Serial Logs](78-bmp180-pressure-sensor-serial.md)
- [80 - BMP180 Weather Trend LCD](80-bmp180-weather-trend-lcd.md)
- [73 - DHT22 Temperature LCD HUD](73-dht22-temperature-lcd-hud.md)
