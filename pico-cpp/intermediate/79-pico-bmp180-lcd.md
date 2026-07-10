# 79 - Pico BMP180 LCD

Display barometric temperature and relative altitude data from a BMP180 sensor on a 16x2 LCD.

## Goal
Learn how to display multi-variable telemetry data from I2C sensors on character displays.

## What You Will Build
An altimeter display panel:
- **BMP180 Sensor (GP4 SDA, GP5 SCL)**: Measures temperature and altitude.
- **16x2 I2C LCD**: Displays current temperature on row 0 (e.g. "T: 24.5 C") and altitude on row 1 (e.g. "Alt: 152.3 m").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| BMP180 Barometric Sensor | `bmp180` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| BMP180 | VCC | 3.3V | Power supply |
| BMP180 | SDA | GP4 | Shared I2C Data |
| BMP180 | SCL | GP5 | Shared I2C Clock |
| BMP180 | GND | GND | Ground return |
| I2C LCD | VCC | 5V | LCD Power |
| I2C LCD | SDA | GP4 | Shared I2C Data |
| I2C LCD | SCL | GP5 | Shared I2C Clock |
| I2C LCD | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_BMP085.h>
#include <LiquidCrystal_I2C.h>

Adafruit_BMP085 bmp;
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  lcd.init();
  lcd.backlight();
  
  if (!bmp.begin()) {
    lcd.setCursor(0, 0);
    lcd.print("Sensor Error!");
  } else {
    lcd.setCursor(0, 0);
    lcd.print("BMP180 Ready");
    delay(1000);
  }
}

void loop() {
  float temp = bmp.readTemperature();
  float altitude = bmp.readAltitude();

  lcd.clear();

  // Row 0: Temperature
  lcd.setCursor(0, 0);
  lcd.print("Temp: ");
  lcd.print(temp);
  lcd.print(" C");

  // Row 1: Altitude
  lcd.setCursor(0, 1);
  lcd.print("Alt : ");
  lcd.print(altitude);
  lcd.print(" m");

  delay(1000); // Update once per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **BMP180 Sensor**, and **I2C LCD** onto the canvas.
2. Connect both BMP180 and LCD to GP4 (SDA) and GP5 (SCL) in parallel.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the sensor pressure slider and observe the temperature and altitude updates.

## Expected Output

Terminal:
```
Simulation active. LCD rendering BMP180 altimeter values.
```

## Expected Canvas Behavior
* Row 0: `Temp: 24.50 C`
* Row 1: `Alt : 112.50 m`

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Wire.begin()` | Joins the master controller to the shared I2C bus network. |

## Hardware & Safety Concept: Shared I2C Bus Architecture
The I2C protocol is a **shared bus** system, meaning multiple devices (such as the BMP180 sensor and the LCD display) connect to the exact same physical SDA and SCL wires. To communicate without conflicts, each device has a unique, hardwired **I2C address** (e.g. 0x77 for the BMP180, and 0x27 for the LCD). The Pico starts communication by sending the specific address byte, ensuring only the target device responds.

## Try This! (Challenges)
1. **Pressure Display**: Modify the code to alternate display views: show temperature/altitude for 3 seconds, then show raw pressure in Pascals for 3 seconds.
2. **Alert Trigger**: Sound a buzzer on GP10 if the altitude drops below 10 meters.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD display shows "Sensor Error!" | BMP180 address conflict | Check that the BMP180 sensor shares the exact same SDA/SCL lines as the LCD display. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [58 - Pico LCD Print](58-pico-lcd-print.md)
- [78 - Pico BMP180 Serial](78-pico-bmp180-serial.md)
